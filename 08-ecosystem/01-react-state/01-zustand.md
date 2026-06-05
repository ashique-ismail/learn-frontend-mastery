# Zustand - Minimal State Management

## The Idea

**In plain English:** Zustand is a tool that helps a React app remember and share information (called "state") across different parts of the page — like how many items are in a shopping cart — without a lot of complicated setup. "State" just means any data your app needs to keep track of and display.

**Real-world analogy:** Imagine a whiteboard in a school common room that everyone can see and write on. Any student can walk up, read what's on the board, erase a number, and write a new one — and everyone in the room instantly sees the update.

- The whiteboard = the Zustand store (the single place all shared data lives)
- The numbers and words written on the board = the state (the actual data being tracked)
- A student reading the board = a React component using a selector to get a value
- A student erasing and rewriting a number = an action calling `set()` to update the state

---

## Overview

Zustand is a small, fast, and scalable state management solution for React. It provides a minimalist API with a Hooks-based interface, no boilerplate, and no Context Provider wrapping required. The name "Zustand" is German for "state."

### Key Features

- **Minimal API**: Simple and intuitive
- **No boilerplate**: Create stores with minimal code
- **No Context Provider**: Access state anywhere
- **TypeScript support**: Full type inference
- **DevTools integration**: Redux DevTools compatible
- **Middleware support**: Persist, immer, devtools
- **Small bundle size**: ~1KB gzipped

## Installation and Setup

```bash
# npm
npm install zustand

# yarn
yarn add zustand

# pnpm
pnpm add zustand
```

## Basic Store Creation

### Simple Counter Store

```typescript
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Usage in component
function Counter() {
  const { count, increment, decrement, reset } = useCounterStore();

  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

### Using Selectors for Performance

```typescript
import { create } from 'zustand';

interface UserStore {
  user: {
    name: string;
    email: string;
    age: number;
  };
  updateName: (name: string) => void;
  updateEmail: (email: string) => void;
}

const useUserStore = create<UserStore>((set) => ({
  user: {
    name: 'John Doe',
    email: 'john@example.com',
    age: 30,
  },
  updateName: (name) => set((state) => ({
    user: { ...state.user, name }
  })),
  updateEmail: (email) => set((state) => ({
    user: { ...state.user, email }
  })),
}));

// Only re-renders when name changes
function UserName() {
  const name = useUserStore((state) => state.user.name);
  return <div>Name: {name}</div>;
}

// Only re-renders when email changes
function UserEmail() {
  const email = useUserStore((state) => state.user.email);
  return <div>Email: {email}</div>;
}
```

## Advanced Selectors

### Shallow Equality Comparison

```typescript
import { create } from 'zustand';
import { shallow } from 'zustand/shallow';

interface TodoStore {
  todos: Array<{ id: string; text: string; completed: boolean }>;
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
}

const useTodoStore = create<TodoStore>((set) => ({
  todos: [],
  addTodo: (text) => set((state) => ({
    todos: [...state.todos, { id: crypto.randomUUID(), text, completed: false }]
  })),
  toggleTodo: (id) => set((state) => ({
    todos: state.todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    )
  })),
}));

// Using shallow comparison for arrays/objects
function TodoList() {
  const todos = useTodoStore((state) => state.todos, shallow);
  
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}
```

### Custom Selectors

```typescript
import { create } from 'zustand';

interface TaskStore {
  tasks: Array<{ id: string; title: string; priority: 'low' | 'medium' | 'high' }>;
  addTask: (title: string, priority: 'low' | 'medium' | 'high') => void;
}

const useTaskStore = create<TaskStore>((set) => ({
  tasks: [],
  addTask: (title, priority) => set((state) => ({
    tasks: [...state.tasks, { id: crypto.randomUUID(), title, priority }]
  })),
}));

// Custom selector for high priority tasks
const useHighPriorityTasks = () => 
  useTaskStore((state) => state.tasks.filter(task => task.priority === 'high'));

// Custom selector with derived state
const useTaskStats = () => 
  useTaskStore((state) => ({
    total: state.tasks.length,
    high: state.tasks.filter(t => t.priority === 'high').length,
    medium: state.tasks.filter(t => t.priority === 'medium').length,
    low: state.tasks.filter(t => t.priority === 'low').length,
  }));

function TaskDashboard() {
  const stats = useTaskStats();
  const highPriorityTasks = useHighPriorityTasks();
  
  return (
    <div>
      <h2>Task Statistics</h2>
      <p>Total: {stats.total}</p>
      <p>High Priority: {stats.high}</p>
      <h3>High Priority Tasks:</h3>
      <ul>
        {highPriorityTasks.map(task => (
          <li key={task.id}>{task.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Async Actions

### Fetching Data

```typescript
import { create } from 'zustand';

interface Product {
  id: string;
  name: string;
  price: number;
}

interface ProductStore {
  products: Product[];
  loading: boolean;
  error: string | null;
  fetchProducts: () => Promise<void>;
  clearError: () => void;
}

const useProductStore = create<ProductStore>((set) => ({
  products: [],
  loading: false,
  error: null,
  
  fetchProducts: async () => {
    set({ loading: true, error: null });
    try {
      const response = await fetch('/api/products');
      if (!response.ok) throw new Error('Failed to fetch products');
      const products = await response.json();
      set({ products, loading: false });
    } catch (error) {
      set({ error: (error as Error).message, loading: false });
    }
  },
  
  clearError: () => set({ error: null }),
}));

function ProductList() {
  const { products, loading, error, fetchProducts, clearError } = useProductStore();

  React.useEffect(() => {
    fetchProducts();
  }, [fetchProducts]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error} <button onClick={clearError}>Clear</button></div>;

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.name} - ${product.price}</li>
      ))}
    </ul>
  );
}
```

### Optimistic Updates

```typescript
import { create } from 'zustand';

interface Comment {
  id: string;
  text: string;
  optimistic?: boolean;
}

interface CommentStore {
  comments: Comment[];
  addComment: (text: string) => Promise<void>;
  deleteComment: (id: string) => Promise<void>;
}

const useCommentStore = create<CommentStore>((set, get) => ({
  comments: [],
  
  addComment: async (text) => {
    const tempId = `temp-${Date.now()}`;
    const optimisticComment: Comment = { id: tempId, text, optimistic: true };
    
    // Add optimistic comment immediately
    set((state) => ({
      comments: [...state.comments, optimisticComment]
    }));
    
    try {
      const response = await fetch('/api/comments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text }),
      });
      const savedComment = await response.json();
      
      // Replace optimistic comment with real one
      set((state) => ({
        comments: state.comments.map(c => 
          c.id === tempId ? savedComment : c
        )
      }));
    } catch (error) {
      // Remove optimistic comment on failure
      set((state) => ({
        comments: state.comments.filter(c => c.id !== tempId)
      }));
      console.error('Failed to add comment:', error);
    }
  },
  
  deleteComment: async (id) => {
    const previousComments = get().comments;
    
    // Optimistically remove comment
    set((state) => ({
      comments: state.comments.filter(c => c.id !== id)
    }));
    
    try {
      await fetch(`/api/comments/${id}`, { method: 'DELETE' });
    } catch (error) {
      // Restore comment on failure
      set({ comments: previousComments });
      console.error('Failed to delete comment:', error);
    }
  },
}));
```

## Middleware

### DevTools Middleware

```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
}

const useCounterStore = create<CounterState>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 }), false, 'increment'),
      decrement: () => set((state) => ({ count: state.count - 1 }), false, 'decrement'),
    }),
    { name: 'CounterStore' }
  )
);
```

### Persist Middleware

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface Settings {
  theme: 'light' | 'dark';
  language: string;
  notifications: boolean;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (language: string) => void;
  toggleNotifications: () => void;
}

const useSettingsStore = create<Settings>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      notifications: true,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      toggleNotifications: () => set((state) => ({ 
        notifications: !state.notifications 
      })),
    }),
    {
      name: 'app-settings',
      storage: createJSONStorage(() => localStorage),
    }
  )
);

// Custom storage implementation
const customStorage = {
  getItem: (name: string) => {
    const str = sessionStorage.getItem(name);
    return str ? JSON.parse(str) : null;
  },
  setItem: (name: string, value: any) => {
    sessionStorage.setItem(name, JSON.stringify(value));
  },
  removeItem: (name: string) => {
    sessionStorage.removeItem(name);
  },
};

const useSessionStore = create<Settings>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      notifications: true,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      toggleNotifications: () => set((state) => ({ 
        notifications: !state.notifications 
      })),
    }),
    {
      name: 'session-settings',
      storage: createJSONStorage(() => customStorage),
    }
  )
);
```

### Immer Middleware

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface NestedState {
  user: {
    profile: {
      name: string;
      address: {
        street: string;
        city: string;
        country: string;
      };
    };
    preferences: {
      theme: string;
      notifications: boolean;
    };
  };
  updateAddress: (updates: Partial<NestedState['user']['profile']['address']>) => void;
}

const useNestedStore = create<NestedState>()(
  immer((set) => ({
    user: {
      profile: {
        name: 'John Doe',
        address: {
          street: '123 Main St',
          city: 'New York',
          country: 'USA',
        },
      },
      preferences: {
        theme: 'light',
        notifications: true,
      },
    },
    
    // Immer allows direct mutation
    updateAddress: (updates) => set((state) => {
      Object.assign(state.user.profile.address, updates);
    }),
  }))
);
```

### Combining Multiple Middleware

```typescript
import { create } from 'zustand';
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface AppStore {
  count: number;
  user: { name: string; email: string };
  increment: () => void;
  updateUser: (updates: Partial<AppStore['user']>) => void;
}

const useAppStore = create<AppStore>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set) => ({
          count: 0,
          user: { name: '', email: '' },
          increment: () => set((state) => {
            state.count++;
          }),
          updateUser: (updates) => set((state) => {
            Object.assign(state.user, updates);
          }),
        }))
      ),
      { name: 'app-store' }
    ),
    { name: 'AppStore' }
  )
);
```

## Slices Pattern

### Creating Modular Stores

```typescript
import { create, StateCreator } from 'zustand';

// User slice
interface UserSlice {
  user: { name: string; email: string } | null;
  login: (name: string, email: string) => void;
  logout: () => void;
}

const createUserSlice: StateCreator<UserSlice & CartSlice, [], [], UserSlice> = (set) => ({
  user: null,
  login: (name, email) => set({ user: { name, email } }),
  logout: () => set({ user: null }),
});

// Cart slice
interface CartSlice {
  items: Array<{ id: string; name: string; quantity: number }>;
  addItem: (item: { id: string; name: string }) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

const createCartSlice: StateCreator<UserSlice & CartSlice, [], [], CartSlice> = (set) => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, { ...item, quantity: 1 }]
  })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter(item => item.id !== id)
  })),
  clearCart: () => set({ items: [] }),
});

// Combined store
const useStore = create<UserSlice & CartSlice>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
}));

function App() {
  const { user, login, logout, items, addItem, clearCart } = useStore();
  
  return (
    <div>
      {user ? (
        <>
          <p>Welcome, {user.name}!</p>
          <button onClick={logout}>Logout</button>
          <p>Cart items: {items.length}</p>
          <button onClick={clearCart}>Clear Cart</button>
        </>
      ) : (
        <button onClick={() => login('John', 'john@example.com')}>Login</button>
      )}
    </div>
  );
}
```

## Subscriptions

### Subscribe to State Changes

```typescript
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

interface NotificationStore {
  message: string | null;
  type: 'info' | 'success' | 'error' | null;
  setNotification: (message: string, type: 'info' | 'success' | 'error') => void;
  clearNotification: () => void;
}

const useNotificationStore = create<NotificationStore>()(
  subscribeWithSelector((set) => ({
    message: null,
    type: null,
    setNotification: (message, type) => set({ message, type }),
    clearNotification: () => set({ message: null, type: null }),
  }))
);

// Subscribe to specific state changes
useNotificationStore.subscribe(
  (state) => state.message,
  (message, previousMessage) => {
    if (message && message !== previousMessage) {
      console.log('New notification:', message);
      
      // Auto-clear after 3 seconds
      setTimeout(() => {
        useNotificationStore.getState().clearNotification();
      }, 3000);
    }
  }
);

// Subscribe to entire store
const unsubscribe = useNotificationStore.subscribe(
  (state) => {
    console.log('Store updated:', state);
  }
);

// Later, unsubscribe
unsubscribe();
```

## Accessing Store Outside React

```typescript
import { create } from 'zustand';

interface AuthStore {
  token: string | null;
  setToken: (token: string) => void;
  clearToken: () => void;
}

const useAuthStore = create<AuthStore>((set) => ({
  token: null,
  setToken: (token) => set({ token }),
  clearToken: () => set({ token: null }),
}));

// API utility using store outside React
class ApiClient {
  async fetchData(endpoint: string) {
    const token = useAuthStore.getState().token;
    
    const response = await fetch(endpoint, {
      headers: {
        'Authorization': token ? `Bearer ${token}` : '',
      },
    });
    
    if (response.status === 401) {
      useAuthStore.getState().clearToken();
    }
    
    return response.json();
  }
}

// Update store from outside React
function loginUser(token: string) {
  useAuthStore.setState({ token });
}

// Reset entire store
function resetApp() {
  useAuthStore.setState({
    token: null,
  }, true); // true = replace entire state
}
```

## Testing Zustand Stores

```typescript
import { create } from 'zustand';
import { act, renderHook } from '@testing-library/react';

interface TestStore {
  count: number;
  increment: () => void;
  decrement: () => void;
}

const useTestStore = create<TestStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
}));

describe('useTestStore', () => {
  beforeEach(() => {
    // Reset store before each test
    useTestStore.setState({ count: 0 });
  });

  it('should increment count', () => {
    const { result } = renderHook(() => useTestStore());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('should decrement count', () => {
    const { result } = renderHook(() => useTestStore());
    
    act(() => {
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(-1);
  });
});

// Create isolated store for testing
function createTestStore() {
  return create<TestStore>((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
    decrement: () => set((state) => ({ count: state.count - 1 })),
  }));
}

describe('isolated store tests', () => {
  it('should work independently', () => {
    const store = createTestStore();
    
    store.getState().increment();
    expect(store.getState().count).toBe(1);
    
    store.getState().increment();
    expect(store.getState().count).toBe(2);
  });
});
```

## Common Mistakes

1. **Not using selectors**: Always use selectors to prevent unnecessary re-renders
2. **Mutating state directly**: Always return new objects/arrays
3. **Overusing shallow comparison**: Only use when necessary for arrays/objects
4. **Creating store inside component**: Create stores at module level
5. **Not memoizing derived state**: Use custom selectors for computed values

## Best Practices

1. **Keep stores focused**: One store per domain/feature
2. **Use TypeScript**: Full type safety and IntelliSense
3. **Organize with slices**: Break large stores into modular slices
4. **Use middleware wisely**: Only add what you need
5. **Test store logic**: Test state updates independently
6. **Document actions**: Clear function names and comments
7. **Handle async carefully**: Include loading and error states
8. **Use DevTools**: Integrate Redux DevTools for debugging

## When to Use Zustand

### Good Use Cases

- Small to medium applications
- Need simple global state
- Want minimal boilerplate
- TypeScript projects
- Performance-critical applications

### Not Ideal For

- Complex state machines (use XState)
- Time-travel debugging requirements
- Very large enterprise apps with complex workflows

## Interview Questions

1. **How does Zustand differ from Redux?**
   - No boilerplate, no providers, simpler API, smaller bundle size

2. **How do you prevent unnecessary re-renders?**
   - Use selectors to subscribe to specific state slices

3. **Can you use Zustand outside React components?**
   - Yes, using `store.getState()` and `store.setState()`

4. **How do you implement persistence?**
   - Use the persist middleware with localStorage/sessionStorage

5. **What middleware options does Zustand provide?**
   - devtools, persist, immer, subscribeWithSelector, combine

## Key Takeaways

- Zustand provides minimal API with maximum flexibility
- No Context Provider wrapping needed
- Use selectors for performance optimization
- Middleware adds powerful features like persistence and devtools
- Slices pattern helps organize large stores
- Can be used outside React components
- Full TypeScript support with excellent type inference
- Perfect for small to medium applications

## Resources

- [Official Documentation](https://docs.pmnd.rs/zustand)
- [GitHub Repository](https://github.com/pmndrs/zustand)
- [TypeScript Guide](https://docs.pmnd.rs/zustand/guides/typescript)
- [Comparison with other libraries](https://docs.pmnd.rs/zustand/getting-started/comparison)
