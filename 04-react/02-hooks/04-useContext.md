# useContext - Context API and Global State

## The Idea

**In plain English:** useContext is a way for any part of your app to read shared information (like who is logged in, or what color theme is selected) without having to pass that information step-by-step through every layer of your code. Think of it as a shared noticeboard that any component can read directly.

**Real-world analogy:** Imagine a school where the principal posts the daily lunch menu on a central noticeboard in the hallway. Every classroom can simply send a student to read the noticeboard directly — no teacher has to personally carry the menu from room to room.

- The noticeboard = the Context (the central place where shared data lives)
- The principal posting the menu = the Provider (the component that puts data into the context)
- A student reading the noticeboard = useContext (the hook a component uses to read the data directly)

---

## Overview

`useContext` is a React Hook that lets you read and subscribe to context from your component. Context provides a way to pass data through the component tree without having to pass props down manually at every level. It's ideal for global state like themes, user authentication, language preferences, and other data that needs to be accessible by many components at different nesting levels.

## Table of Contents

1. [Basic Concepts](#basic-concepts)
2. [Creating and Using Context](#creating-and-using-context)
3. [Context Patterns](#context-patterns)
4. [Advanced Use Cases](#advanced-use-cases)
5. [Performance Optimization](#performance-optimization)
6. [Best Practices](#best-practices)
7. [Common Pitfalls](#common-pitfalls)
8. [Real-World Examples](#real-world-examples)

## Basic Concepts

### The Prop Drilling Problem

```jsx
// ❌ Without Context: Prop Drilling
function App() {
  const [user, setUser] = useState({ name: 'John', role: 'admin' });
  
  return <Dashboard user={user} />;
}

function Dashboard({ user }) {
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {
  return <Navigation user={user} />;
}

function Navigation({ user }) {
  return <UserMenu user={user} />;
}

function UserMenu({ user }) {
  return <div>{user.name}</div>; // Finally used here!
}

// User prop passed through 4 components that don't use it!
```

### Context Solution

```jsx
import { createContext, useContext, useState } from 'react';

// ✅ With Context: Direct Access
const UserContext = createContext(null);

function App() {
  const [user, setUser] = useState({ name: 'John', role: 'admin' });
  
  return (
    <UserContext.Provider value={user}>
      <Dashboard />
    </UserContext.Provider>
  );
}

function Dashboard() {
  return <Sidebar />; // No props needed
}

function Sidebar() {
  return <Navigation />; // No props needed
}

function Navigation() {
  return <UserMenu />; // No props needed
}

function UserMenu() {
  const user = useContext(UserContext); // Access directly
  return <div>{user.name}</div>;
}
```

### Basic useContext Syntax

```jsx
import { createContext, useContext } from 'react';

// 1. Create context
const MyContext = createContext(defaultValue);

// 2. Provide context value
function Parent() {
  return (
    <MyContext.Provider value={someValue}>
      <Child />
    </MyContext.Provider>
  );
}

// 3. Consume context
function Child() {
  const value = useContext(MyContext);
  return <div>{value}</div>;
}
```

## Creating and Using Context

### Simple Context Example

```jsx
import { createContext, useContext, useState } from 'react';

// Create context with default value
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Toolbar />
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </ThemeContext.Provider>
  );
}

function Toolbar() {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext);
  
  return (
    <button className={theme}>
      I am styled by {theme} theme
    </button>
  );
}
```

### Context with Complex State

```jsx
const UserContext = createContext(null);

function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Fetch user on mount
    fetchCurrentUser()
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);
  
  const login = async (credentials) => {
    const user = await loginAPI(credentials);
    setUser(user);
  };
  
  const logout = () => {
    setUser(null);
    logoutAPI();
  };
  
  const value = {
    user,
    loading,
    login,
    logout
  };
  
  return (
    <UserContext.Provider value={value}>
      {children}
    </UserContext.Provider>
  );
}

// Custom hook for easier usage
function useUser() {
  const context = useContext(UserContext);
  
  if (context === undefined) {
    throw new Error('useUser must be used within UserProvider');
  }
  
  return context;
}

// Usage
function App() {
  return (
    <UserProvider>
      <Dashboard />
    </UserProvider>
  );
}

function Dashboard() {
  const { user, loading, logout } = useUser();
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div>
      <h1>Welcome, {user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### Multiple Contexts

```jsx
const ThemeContext = createContext('light');
const UserContext = createContext(null);
const LanguageContext = createContext('en');

function App() {
  const [theme, setTheme] = useState('light');
  const [user, setUser] = useState(null);
  const [language, setLanguage] = useState('en');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <UserContext.Provider value={{ user, setUser }}>
        <LanguageContext.Provider value={{ language, setLanguage }}>
          <MainApp />
        </LanguageContext.Provider>
      </UserContext.Provider>
    </ThemeContext.Provider>
  );
}

function MainApp() {
  const { theme } = useContext(ThemeContext);
  const { user } = useContext(UserContext);
  const { language } = useContext(LanguageContext);
  
  return (
    <div className={theme}>
      <p>{language === 'en' ? 'Hello' : 'Hola'}, {user?.name}</p>
    </div>
  );
}
```

## Context Patterns

### Provider Pattern with Custom Hook

```jsx
import { createContext, useContext, useState } from 'react';

const CartContext = createContext(undefined);

export function CartProvider({ children }) {
  const [items, setItems] = useState([]);
  
  const addItem = (item) => {
    setItems(prev => [...prev, item]);
  };
  
  const removeItem = (id) => {
    setItems(prev => prev.filter(item => item.id !== id));
  };
  
  const clearCart = () => {
    setItems([]);
  };
  
  const total = items.reduce((sum, item) => sum + item.price, 0);
  
  const value = {
    items,
    addItem,
    removeItem,
    clearCart,
    total
  };
  
  return (
    <CartContext.Provider value={value}>
      {children}
    </CartContext.Provider>
  );
}

// Custom hook with error checking
export function useCart() {
  const context = useContext(CartContext);
  
  if (context === undefined) {
    throw new Error('useCart must be used within CartProvider');
  }
  
  return context;
}

// Usage
function App() {
  return (
    <CartProvider>
      <ShoppingApp />
    </CartProvider>
  );
}

function ShoppingApp() {
  const { items, total, addItem } = useCart();
  
  return (
    <div>
      <h2>Cart Total: ${total}</h2>
      <p>Items: {items.length}</p>
      <button onClick={() => addItem({ id: 1, name: 'Product', price: 10 })}>
        Add Item
      </button>
    </div>
  );
}
```

### Compound Context Pattern

```jsx
const TabsContext = createContext(null);

function Tabs({ children, defaultValue }) {
  const [activeTab, setActiveTab] = useState(defaultValue);
  
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div className="tab-list">{children}</div>;
}

function Tab({ value, children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  const isActive = activeTab === value;
  
  return (
    <button
      className={isActive ? 'active' : ''}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }) {
  const { activeTab } = useContext(TabsContext);
  
  if (activeTab !== value) return null;
  
  return <div className="tab-panel">{children}</div>;
}

// Usage
function App() {
  return (
    <Tabs defaultValue="tab1">
      <TabList>
        <Tab value="tab1">First</Tab>
        <Tab value="tab2">Second</Tab>
        <Tab value="tab3">Third</Tab>
      </TabList>
      
      <TabPanel value="tab1">
        <p>First tab content</p>
      </TabPanel>
      <TabPanel value="tab2">
        <p>Second tab content</p>
      </TabPanel>
      <TabPanel value="tab3">
        <p>Third tab content</p>
      </TabPanel>
    </Tabs>
  );
}
```

### Reducer Pattern with Context

```jsx
import { createContext, useContext, useReducer } from 'react';

const TodoContext = createContext(null);

const todoReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return [...state, { id: Date.now(), text: action.text, done: false }];
    case 'TOGGLE_TODO':
      return state.map(todo =>
        todo.id === action.id ? { ...todo, done: !todo.done } : todo
      );
    case 'DELETE_TODO':
      return state.filter(todo => todo.id !== action.id);
    default:
      return state;
  }
};

export function TodoProvider({ children }) {
  const [todos, dispatch] = useReducer(todoReducer, []);
  
  return (
    <TodoContext.Provider value={{ todos, dispatch }}>
      {children}
    </TodoContext.Provider>
  );
}

export function useTodos() {
  const context = useContext(TodoContext);
  if (!context) {
    throw new Error('useTodos must be used within TodoProvider');
  }
  return context;
}

// Usage
function TodoApp() {
  const { todos, dispatch } = useTodos();
  const [text, setText] = useState('');
  
  const addTodo = () => {
    dispatch({ type: 'ADD_TODO', text });
    setText('');
  };
  
  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={addTodo}>Add</button>
      
      {todos.map(todo => (
        <div key={todo.id}>
          <span
            style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
            onClick={() => dispatch({ type: 'TOGGLE_TODO', id: todo.id })}
          >
            {todo.text}
          </span>
          <button onClick={() => dispatch({ type: 'DELETE_TODO', id: todo.id })}>
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}
```

### Factory Pattern for Multiple Instances

```jsx
function createContextProvider(name) {
  const Context = createContext(undefined);
  
  function Provider({ children, value }) {
    return <Context.Provider value={value}>{children}</Context.Provider>;
  }
  
  function useValue() {
    const context = useContext(Context);
    if (context === undefined) {
      throw new Error(`use${name} must be used within ${name}Provider`);
    }
    return context;
  }
  
  return [Provider, useValue];
}

// Create multiple context instances
const [ThemeProvider, useTheme] = createContextProvider('Theme');
const [AuthProvider, useAuth] = createContextProvider('Auth');
const [CartProvider, useCart] = createContextProvider('Cart');

// Usage
function App() {
  const [theme, setTheme] = useState('light');
  const [user, setUser] = useState(null);
  const [cart, setCart] = useState([]);
  
  return (
    <ThemeProvider value={{ theme, setTheme }}>
      <AuthProvider value={{ user, setUser }}>
        <CartProvider value={{ cart, setCart }}>
          <MainApp />
        </CartProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}
```

## Advanced Use Cases

### Context with Local Storage Sync

```jsx
const SettingsContext = createContext(null);

export function SettingsProvider({ children }) {
  const [settings, setSettings] = useState(() => {
    const saved = localStorage.getItem('settings');
    return saved ? JSON.parse(saved) : {
      theme: 'light',
      language: 'en',
      notifications: true
    };
  });
  
  useEffect(() => {
    localStorage.setItem('settings', JSON.stringify(settings));
  }, [settings]);
  
  const updateSetting = (key, value) => {
    setSettings(prev => ({ ...prev, [key]: value }));
  };
  
  return (
    <SettingsContext.Provider value={{ settings, updateSetting }}>
      {children}
    </SettingsContext.Provider>
  );
}

export function useSettings() {
  const context = useContext(SettingsContext);
  if (!context) {
    throw new Error('useSettings must be used within SettingsProvider');
  }
  return context;
}
```

### Context with Async Initialization

```jsx
const DataContext = createContext(null);

export function DataProvider({ children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchData() {
      try {
        setLoading(true);
        const response = await fetch('/api/data');
        const result = await response.json();
        
        if (!cancelled) {
          setData(result);
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
  }, []);
  
  const refetch = useCallback(async () => {
    setLoading(true);
    try {
      const response = await fetch('/api/data');
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, []);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <DataContext.Provider value={{ data, refetch }}>
      {children}
    </DataContext.Provider>
  );
}
```

### Context with Middleware

```jsx
const StoreContext = createContext(null);

function logger(prevState, action, nextState) {
  console.log('Previous State:', prevState);
  console.log('Action:', action);
  console.log('Next State:', nextState);
}

function analytics(prevState, action, nextState) {
  if (action.type === 'USER_ACTION') {
    trackEvent('user_action', { action });
  }
}

export function StoreProvider({ children, middleware = [] }) {
  const [state, setState] = useState({ count: 0 });
  
  const dispatch = (action) => {
    const prevState = state;
    
    // Handle action
    let nextState;
    switch (action.type) {
      case 'INCREMENT':
        nextState = { ...state, count: state.count + 1 };
        break;
      case 'DECREMENT':
        nextState = { ...state, count: state.count - 1 };
        break;
      default:
        nextState = state;
    }
    
    // Run middleware
    middleware.forEach(fn => fn(prevState, action, nextState));
    
    setState(nextState);
  };
  
  return (
    <StoreContext.Provider value={{ state, dispatch }}>
      {children}
    </StoreContext.Provider>
  );
}

// Usage with middleware
function App() {
  return (
    <StoreProvider middleware={[logger, analytics]}>
      <Counter />
    </StoreProvider>
  );
}
```

### Nested Contexts with Composition

```jsx
// Feature-specific contexts
const AuthContext = createContext(null);
const ThemeContext = createContext(null);
const NotificationContext = createContext(null);

// Combine into single provider
export function AppProvider({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <NotificationProvider>
          {children}
        </NotificationProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

// Or use array composition
function composeProviders(...providers) {
  return ({ children }) => {
    return providers.reduceRight((acc, Provider) => {
      return <Provider>{acc}</Provider>;
    }, children);
  };
}

const AppProvider = composeProviders(
  AuthProvider,
  ThemeProvider,
  NotificationProvider
);

// Usage
function App() {
  return (
    <AppProvider>
      <MainApp />
    </AppProvider>
  );
}
```

## Performance Optimization

### Splitting Context for Better Performance

```jsx
// ❌ BAD: Single context with all values
const AppContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [settings, setSettings] = useState({});
  
  const value = {
    user, setUser,
    theme, setTheme,
    settings, setSettings
  };
  
  // Changing any value re-renders ALL consumers
  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}

// ✅ GOOD: Split into separate contexts
const UserContext = createContext(null);
const ThemeContext = createContext(null);
const SettingsContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [settings, setSettings] = useState({});
  
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <ThemeContext.Provider value={{ theme, setTheme }}>
        <SettingsContext.Provider value={{ settings, setSettings }}>
          {children}
        </SettingsContext.Provider>
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}

// Now components only re-render when their specific context changes
function UserProfile() {
  const { user } = useContext(UserContext); // Only re-renders on user change
  return <div>{user?.name}</div>;
}

function ThemeButton() {
  const { theme, setTheme } = useContext(ThemeContext); // Only re-renders on theme change
  return <button onClick={() => setTheme('dark')}>{theme}</button>;
}
```

### Memoizing Context Values

```jsx
// ❌ BAD: New object on every render
function BadProvider({ children }) {
  const [user, setUser] = useState(null);
  
  // New object every render = all consumers re-render
  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  );
}

// ✅ GOOD: Memoized value
function GoodProvider({ children }) {
  const [user, setUser] = useState(null);
  
  const value = useMemo(() => ({ user, setUser }), [user]);
  
  return (
    <UserContext.Provider value={value}>
      {children}
    </UserContext.Provider>
  );
}

// ✅ BETTER: Split state and dispatch
const UserStateContext = createContext(null);
const UserDispatchContext = createContext(null);

function BestProvider({ children }) {
  const [user, setUser] = useState(null);
  
  // State context changes when user changes
  // Dispatch context never changes
  return (
    <UserStateContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserStateContext.Provider>
  );
}

function useUserState() {
  return useContext(UserStateContext);
}

function useUserDispatch() {
  return useContext(UserDispatchContext);
}

// Components that only need to set user won't re-render
function LoginForm() {
  const setUser = useUserDispatch(); // Never re-renders
  
  const handleLogin = async (credentials) => {
    const user = await login(credentials);
    setUser(user);
  };
  
  return <form onSubmit={handleLogin}>...</form>;
}

// Components that read user re-render only when user changes
function UserProfile() {
  const user = useUserState(); // Re-renders on user change
  return <div>{user?.name}</div>;
}
```

### Selector Pattern for Partial Updates

```jsx
const StoreContext = createContext(null);

function StoreProvider({ children }) {
  const [store, setStore] = useState({
    user: null,
    posts: [],
    comments: [],
    settings: {}
  });
  
  return (
    <StoreContext.Provider value={{ store, setStore }}>
      {children}
    </StoreContext.Provider>
  );
}

// Custom hook with selector
function useStoreSelector(selector) {
  const { store } = useContext(StoreContext);
  return useMemo(() => selector(store), [store, selector]);
}

// Usage - only re-renders when selected value changes
function UserName() {
  const userName = useStoreSelector(store => store.user?.name);
  return <div>{userName}</div>;
}

function PostCount() {
  const postCount = useStoreSelector(store => store.posts.length);
  return <div>Posts: {postCount}</div>;
}
```

## Best Practices

### 1. Custom Hooks for Context

```jsx
// ✅ Always wrap context in custom hook
const ThemeContext = createContext(undefined);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  
  if (context === undefined) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  
  return context;
}

// Usage is cleaner
function Component() {
  const { theme } = useTheme(); // Instead of useContext(ThemeContext)
  return <div className={theme}>Content</div>;
}
```

### 2. Default Values and Error Handling

```jsx
// Meaningful default value
const ThemeContext = createContext('light');

// Or undefined with error checking
const UserContext = createContext(undefined);

function useUser() {
  const context = useContext(UserContext);
  
  if (context === undefined) {
    throw new Error(
      'useUser must be used within a UserProvider. ' +
      'Wrap a parent component in <UserProvider> to fix this error.'
    );
  }
  
  return context;
}
```

### 3. Keep Context Focused

```jsx
// ❌ BAD: God object context
const AppContext = createContext({
  user: null,
  theme: 'light',
  settings: {},
  cart: [],
  notifications: [],
  // ... too many things
});

// ✅ GOOD: Separate, focused contexts
const UserContext = createContext(null);
const ThemeContext = createContext('light');
const SettingsContext = createContext({});
const CartContext = createContext([]);
const NotificationContext = createContext([]);
```

### 4. Documentation

```jsx
/**
 * AuthContext provides authentication state and methods.
 * 
 * @example
 * function MyComponent() {
 *   const { user, login, logout } = useAuth();
 *   
 *   if (!user) {
 *     return <button onClick={() => login(credentials)}>Login</button>;
 *   }
 *   
 *   return (
 *     <div>
 *       <p>Welcome, {user.name}</p>
 *       <button onClick={logout}>Logout</button>
 *     </div>
 *   );
 * }
 */
const AuthContext = createContext(undefined);

/**
 * Hook to access authentication context.
 * Must be used within AuthProvider.
 * 
 * @returns {Object} Authentication context
 * @returns {User|null} user - Current user or null
 * @returns {Function} login - Login function
 * @returns {Function} logout - Logout function
 * 
 * @throws {Error} If used outside of AuthProvider
 */
export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

## Common Pitfalls

### 1. Unnecessary Re-renders

```jsx
// ❌ BAD: Inline object causes re-renders
function BadProvider({ children }) {
  const [count, setCount] = useState(0);
  
  return (
    <CountContext.Provider value={{ count, setCount }}>
      {children}
    </CountContext.Provider>
  );
}

// ✅ GOOD: Memoized value
function GoodProvider({ children }) {
  const [count, setCount] = useState(0);
  const value = useMemo(() => ({ count, setCount }), [count]);
  
  return (
    <CountContext.Provider value={value}>
      {children}
    </CountContext.Provider>
  );
}
```

### 2. Not Providing Default Value

```jsx
// ❌ BAD: No default, undefined on first render
const ThemeContext = createContext();

// ✅ GOOD: Provide meaningful default
const ThemeContext = createContext('light');

// ✅ ALSO GOOD: undefined with error handling
const UserContext = createContext(undefined);

function useUser() {
  const context = useContext(UserContext);
  if (context === undefined) {
    throw new Error('useUser must be used within UserProvider');
  }
  return context;
}
```

### 3. Context Hell

```jsx
// ❌ BAD: Nested provider pyramid
function App() {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LanguageProvider>
          <NotificationProvider>
            <CartProvider>
              <SettingsProvider>
                <MainApp />
              </SettingsProvider>
            </CartProvider>
          </NotificationProvider>
        </LanguageProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

// ✅ GOOD: Compose providers
function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LanguageProvider>
          <NotificationProvider>
            <CartProvider>
              <SettingsProvider>
                {children}
              </SettingsProvider>
            </CartProvider>
          </NotificationProvider>
        </LanguageProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <MainApp />
    </AppProviders>
  );
}
```

## Real-World Examples

### Complete Authentication System

```jsx
import { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext(undefined);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  // Check if user is logged in on mount
  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      fetchUser(token)
        .then(setUser)
        .catch(() => localStorage.removeItem('token'))
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);
  
  const login = async (email, password) => {
    try {
      setError(null);
      const { user, token } = await loginAPI(email, password);
      localStorage.setItem('token', token);
      setUser(user);
      return user;
    } catch (err) {
      setError(err.message);
      throw err;
    }
  };
  
  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
    logoutAPI();
  };
  
  const signup = async (email, password, name) => {
    try {
      setError(null);
      const { user, token } = await signupAPI(email, password, name);
      localStorage.setItem('token', token);
      setUser(user);
      return user;
    } catch (err) {
      setError(err.message);
      throw err;
    }
  };
  
  const value = useMemo(() => ({
    user,
    loading,
    error,
    login,
    logout,
    signup
  }), [user, loading, error]);
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Protected Route component
export function ProtectedRoute({ children }) {
  const { user, loading } = useAuth();
  
  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  
  return children;
}

// Usage
function App() {
  return (
    <AuthProvider>
      <Router>
        <Routes>
          <Route path="/login" element={<Login />} />
          <Route path="/signup" element={<Signup />} />
          <Route
            path="/dashboard"
            element={
              <ProtectedRoute>
                <Dashboard />
              </ProtectedRoute>
            }
          />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

## Summary

### Key Takeaways

1. **Purpose**: Share data across component tree without prop drilling
2. **Structure**: Create context, provide value, consume with `useContext`
3. **Custom Hooks**: Always wrap context in custom hook for better DX
4. **Performance**: Memoize values, split contexts, use selectors
5. **Best For**: Global app state, theme, auth, language, configuration

### When to Use Context

- Authentication state
- Theme/styling configuration
- Language/i18n settings
- User preferences
- Feature flags
- Global UI state (modals, notifications)

### When NOT to Use Context

- Frequently changing data (use state management library)
- Data needed by few components (use props)
- Performance-critical updates (consider alternatives)
- Complex state logic (consider useReducer + Context or state management library)

Context is powerful but should be used judiciously. For complex state management needs, consider libraries like Redux, Zustand, or Jotai.
