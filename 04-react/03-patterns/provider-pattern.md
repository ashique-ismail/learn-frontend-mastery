# Provider Pattern

## Overview

The **Provider Pattern** uses React Context to make data available to multiple components without prop drilling. A Provider component wraps part of the component tree and provides values to all descendants.

```jsx
<DataProvider>
  <ChildComponent />  {/* Can access data */}
</DataProvider>
```

## Basic Implementation

### Simple Provider

```jsx
import { createContext, useContext, useState } from 'react';

// 1. Create Context
const ThemeContext = createContext(null);

// 2. Create Provider Component
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  const value = {
    theme,
    setTheme,
    toggleTheme: () => setTheme(t => t === 'light' ? 'dark' : 'light')
  };
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. Create Hook for Convenience
export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

// Usage
function App() {
  return (
    <ThemeProvider>
      <Page />
    </ThemeProvider>
  );
}

function Page() {
  const { theme, toggleTheme } = useTheme();
  
  return (
    <div className={theme}>
      <button onClick={toggleTheme}>
        Current theme: {theme}
      </button>
    </div>
  );
}
```

## Practical Examples

### Example 1: Auth Provider

```jsx
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Check if user is logged in
    checkAuth()
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);
  
  const login = async (credentials) => {
    const user = await loginAPI(credentials);
    setUser(user);
  };
  
  const logout = async () => {
    await logoutAPI();
    setUser(null);
  };
  
  const value = {
    user,
    loading,
    login,
    logout,
    isAuthenticated: !!user
  };
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Usage: Login Form
function LoginForm() {
  const { login } = useAuth();
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    await login(credentials);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={credentials.email}
        onChange={(e) => setCredentials({ ...credentials, email: e.target.value })}
      />
      <input
        type="password"
        value={credentials.password}
        onChange={(e) => setCredentials({ ...credentials, password: e.target.value })}
      />
      <button type="submit">Login</button>
    </form>
  );
}

// Usage: Protected Route
function ProtectedRoute({ children }) {
  const { isAuthenticated, loading } = useAuth();
  
  if (loading) return <Spinner />;
  
  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }
  
  return children;
}

// Usage: User Menu
function UserMenu() {
  const { user, logout } = useAuth();
  
  return (
    <div>
      <span>Hello, {user.name}</span>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

// App setup
function App() {
  return (
    <AuthProvider>
      <Router>
        <Routes>
          <Route path="/login" element={<LoginForm />} />
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

### Example 2: Shopping Cart Provider

```jsx
const CartContext = createContext(null);

export function CartProvider({ children }) {
  const [items, setItems] = useState([]);
  
  const addToCart = (product, quantity = 1) => {
    setItems(prevItems => {
      const existingItem = prevItems.find(item => item.id === product.id);
      
      if (existingItem) {
        return prevItems.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + quantity }
            : item
        );
      }
      
      return [...prevItems, { ...product, quantity }];
    });
  };
  
  const removeFromCart = (productId) => {
    setItems(prevItems => prevItems.filter(item => item.id !== productId));
  };
  
  const updateQuantity = (productId, quantity) => {
    if (quantity <= 0) {
      removeFromCart(productId);
      return;
    }
    
    setItems(prevItems =>
      prevItems.map(item =>
        item.id === productId ? { ...item, quantity } : item
      )
    );
  };
  
  const clearCart = () => {
    setItems([]);
  };
  
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const itemCount = items.reduce((sum, item) => sum + item.quantity, 0);
  
  const value = {
    items,
    addToCart,
    removeFromCart,
    updateQuantity,
    clearCart,
    total,
    itemCount
  };
  
  return (
    <CartContext.Provider value={value}>
      {children}
    </CartContext.Provider>
  );
}

export function useCart() {
  const context = useContext(CartContext);
  if (!context) {
    throw new Error('useCart must be used within CartProvider');
  }
  return context;
}

// Usage: Product Card
function ProductCard({ product }) {
  const { addToCart } = useCart();
  
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => addToCart(product)}>
        Add to Cart
      </button>
    </div>
  );
}

// Usage: Cart Widget
function CartWidget() {
  const { itemCount, total } = useCart();
  
  return (
    <div className="cart-widget">
      <span>🛒 {itemCount} items</span>
      <span>${total.toFixed(2)}</span>
    </div>
  );
}

// Usage: Cart Page
function CartPage() {
  const { items, updateQuantity, removeFromCart, clearCart, total } = useCart();
  
  if (items.length === 0) {
    return <div>Your cart is empty</div>;
  }
  
  return (
    <div>
      {items.map(item => (
        <div key={item.id} className="cart-item">
          <h4>{item.name}</h4>
          <p>${item.price} × {item.quantity}</p>
          <input
            type="number"
            value={item.quantity}
            onChange={(e) => updateQuantity(item.id, parseInt(e.target.value))}
          />
          <button onClick={() => removeFromCart(item.id)}>Remove</button>
        </div>
      ))}
      
      <div>
        <h3>Total: ${total.toFixed(2)}</h3>
        <button onClick={clearCart}>Clear Cart</button>
        <button>Checkout</button>
      </div>
    </div>
  );
}
```

### Example 3: Modal Provider

```jsx
const ModalContext = createContext(null);

export function ModalProvider({ children }) {
  const [modals, setModals] = useState([]);
  
  const openModal = (component, props = {}) => {
    const id = Date.now();
    setModals(prev => [...prev, { id, component, props }]);
    return id;
  };
  
  const closeModal = (id) => {
    setModals(prev => prev.filter(modal => modal.id !== id));
  };
  
  const closeAllModals = () => {
    setModals([]);
  };
  
  const value = {
    openModal,
    closeModal,
    closeAllModals
  };
  
  return (
    <ModalContext.Provider value={value}>
      {children}
      
      {/* Render all open modals */}
      {modals.map(({ id, component: Component, props }) => (
        <Component
          key={id}
          {...props}
          onClose={() => closeModal(id)}
        />
      ))}
    </ModalContext.Provider>
  );
}

export function useModal() {
  const context = useContext(ModalContext);
  if (!context) {
    throw new Error('useModal must be used within ModalProvider');
  }
  return context;
}

// Modal Components
function ConfirmModal({ onClose, onConfirm, message }) {
  return (
    <div className="modal-overlay">
      <div className="modal">
        <p>{message}</p>
        <button onClick={() => { onConfirm(); onClose(); }}>
          Confirm
        </button>
        <button onClick={onClose}>Cancel</button>
      </div>
    </div>
  );
}

function AlertModal({ onClose, message }) {
  return (
    <div className="modal-overlay">
      <div className="modal">
        <p>{message}</p>
        <button onClick={onClose}>OK</button>
      </div>
    </div>
  );
}

// Usage
function DeleteButton({ itemId }) {
  const { openModal } = useModal();
  
  const handleDelete = () => {
    openModal(ConfirmModal, {
      message: 'Are you sure you want to delete this item?',
      onConfirm: async () => {
        await deleteItem(itemId);
        openModal(AlertModal, {
          message: 'Item deleted successfully'
        });
      }
    });
  };
  
  return <button onClick={handleDelete}>Delete</button>;
}
```

### Example 4: Toast/Notification Provider

```jsx
const ToastContext = createContext(null);

export function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);
  
  const addToast = (message, type = 'info', duration = 3000) => {
    const id = Date.now();
    const toast = { id, message, type };
    
    setToasts(prev => [...prev, toast]);
    
    if (duration) {
      setTimeout(() => {
        removeToast(id);
      }, duration);
    }
    
    return id;
  };
  
  const removeToast = (id) => {
    setToasts(prev => prev.filter(toast => toast.id !== id));
  };
  
  const success = (message, duration) => addToast(message, 'success', duration);
  const error = (message, duration) => addToast(message, 'error', duration);
  const info = (message, duration) => addToast(message, 'info', duration);
  const warning = (message, duration) => addToast(message, 'warning', duration);
  
  const value = {
    addToast,
    removeToast,
    success,
    error,
    info,
    warning
  };
  
  return (
    <ToastContext.Provider value={value}>
      {children}
      
      <div className="toast-container">
        {toasts.map(toast => (
          <div
            key={toast.id}
            className={`toast toast-${toast.type}`}
            onClick={() => removeToast(toast.id)}
          >
            {toast.message}
          </div>
        ))}
      </div>
    </ToastContext.Provider>
  );
}

export function useToast() {
  const context = useContext(ToastContext);
  if (!context) {
    throw new Error('useToast must be used within ToastProvider');
  }
  return context;
}

// Usage
function SaveButton({ data }) {
  const { success, error } = useToast();
  
  const handleSave = async () => {
    try {
      await saveData(data);
      success('Data saved successfully!');
    } catch (err) {
      error('Failed to save data');
    }
  };
  
  return <button onClick={handleSave}>Save</button>;
}
```

### Example 5: Settings Provider with LocalStorage

```jsx
const SettingsContext = createContext(null);

export function SettingsProvider({ children }) {
  const [settings, setSettings] = useState(() => {
    const stored = localStorage.getItem('settings');
    return stored ? JSON.parse(stored) : {
      theme: 'light',
      language: 'en',
      notifications: true,
      fontSize: 'medium'
    };
  });
  
  useEffect(() => {
    localStorage.setItem('settings', JSON.stringify(settings));
  }, [settings]);
  
  const updateSettings = (updates) => {
    setSettings(prev => ({ ...prev, ...updates }));
  };
  
  const resetSettings = () => {
    setSettings({
      theme: 'light',
      language: 'en',
      notifications: true,
      fontSize: 'medium'
    });
  };
  
  const value = {
    settings,
    updateSettings,
    resetSettings
  };
  
  return (
    <SettingsContext.Provider value={value}>
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

// Usage
function SettingsPage() {
  const { settings, updateSettings, resetSettings } = useSettings();
  
  return (
    <div>
      <h1>Settings</h1>
      
      <label>
        Theme:
        <select
          value={settings.theme}
          onChange={(e) => updateSettings({ theme: e.target.value })}
        >
          <option value="light">Light</option>
          <option value="dark">Dark</option>
        </select>
      </label>
      
      <label>
        Notifications:
        <input
          type="checkbox"
          checked={settings.notifications}
          onChange={(e) => updateSettings({ notifications: e.target.checked })}
        />
      </label>
      
      <button onClick={resetSettings}>Reset to Defaults</button>
    </div>
  );
}
```

## Advanced Patterns

### Composing Multiple Providers

```jsx
function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <SettingsProvider>
          <CartProvider>
            <ToastProvider>
              <ModalProvider>
                {children}
              </ModalProvider>
            </ToastProvider>
          </CartProvider>
        </SettingsProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <Router>
        <Routes>
          {/* Your routes */}
        </Routes>
      </Router>
    </AppProviders>
  );
}
```

### Provider with Reducer

```jsx
const initialState = {
  user: null,
  loading: false,
  error: null
};

function authReducer(state, action) {
  switch (action.type) {
    case 'LOGIN_START':
      return { ...state, loading: true, error: null };
    case 'LOGIN_SUCCESS':
      return { ...state, user: action.payload, loading: false };
    case 'LOGIN_ERROR':
      return { ...state, error: action.payload, loading: false };
    case 'LOGOUT':
      return { ...initialState };
    default:
      return state;
  }
}

export function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, initialState);
  
  const login = async (credentials) => {
    dispatch({ type: 'LOGIN_START' });
    try {
      const user = await loginAPI(credentials);
      dispatch({ type: 'LOGIN_SUCCESS', payload: user });
    } catch (error) {
      dispatch({ type: 'LOGIN_ERROR', payload: error.message });
    }
  };
  
  const logout = () => {
    dispatch({ type: 'LOGOUT' });
  };
  
  const value = {
    ...state,
    login,
    logout
  };
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}
```

### Optimized Provider (Preventing Re-renders)

```jsx
// Problem: Value object causes re-renders
export function BadProvider({ children }) {
  const [count, setCount] = useState(0);
  
  // New object every render!
  const value = { count, setCount };
  
  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  );
}

// Solution: useMemo
export function GoodProvider({ children }) {
  const [count, setCount] = useState(0);
  
  const value = useMemo(() => ({
    count,
    setCount
  }), [count]);
  
  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  );
}

// Better: Split contexts
const CountContext = createContext(0);
const CountDispatchContext = createContext(() => {});

export function BestProvider({ children }) {
  const [count, setCount] = useState(0);
  
  return (
    <CountContext.Provider value={count}>
      <CountDispatchContext.Provider value={setCount}>
        {children}
      </CountDispatchContext.Provider>
    </CountContext.Provider>
  );
}

// Consumers only re-render when they use changes
function DisplayCount() {
  const count = useContext(CountContext);  // Re-renders on count change
  return <div>{count}</div>;
}

function IncrementButton() {
  const setCount = useContext(CountDispatchContext);  // Never re-renders
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

## TypeScript

```tsx
interface AuthContextType {
  user: User | null;
  loading: boolean;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => Promise<void>;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);
  
  const login = async (credentials: Credentials) => {
    setLoading(true);
    try {
      const user = await loginAPI(credentials);
      setUser(user);
    } finally {
      setLoading(false);
    }
  };
  
  const logout = async () => {
    await logoutAPI();
    setUser(null);
  };
  
  const value: AuthContextType = {
    user,
    loading,
    login,
    logout,
    isAuthenticated: !!user
  };
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

## Testing

```jsx
import { render, screen } from '@testing-library/react';
import { AuthProvider, useAuth } from './AuthProvider';

// Helper component for testing
function TestComponent() {
  const { user, login } = useAuth();
  
  return (
    <div>
      <div>{user ? user.name : 'Not logged in'}</div>
      <button onClick={() => login({ email: 'test@example.com', password: '123' })}>
        Login
      </button>
    </div>
  );
}

test('provides auth context', async () => {
  render(
    <AuthProvider>
      <TestComponent />
    </AuthProvider>
  );
  
  expect(screen.getByText('Not logged in')).toBeInTheDocument();
  
  fireEvent.click(screen.getByText('Login'));
  
  await waitFor(() => {
    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });
});

// Test error handling
test('throws error when used outside provider', () => {
  // Suppress console.error for this test
  const spy = jest.spyOn(console, 'error').mockImplementation(() => {});
  
  expect(() => {
    render(<TestComponent />);
  }).toThrow('useAuth must be used within AuthProvider');
  
  spy.mockRestore();
});
```

## Best Practices

1. **Split by feature**: Create separate providers for unrelated features
2. **Memoize values**: Use useMemo to prevent unnecessary re-renders
3. **Custom hooks**: Always provide a custom hook for cleaner API
4. **Error handling**: Throw error if hook used outside provider
5. **Default values**: Provide sensible defaults in createContext
6. **Split read/write**: Separate state and dispatch contexts for optimization

## Resources

- [React Context](https://react.dev/learn/passing-data-deeply-with-context)
- [Context Performance](https://github.com/facebook/react/issues/15156#issuecomment-474590693)
- [When to Use Context](https://kentcdodds.com/blog/application-state-management-with-react)
