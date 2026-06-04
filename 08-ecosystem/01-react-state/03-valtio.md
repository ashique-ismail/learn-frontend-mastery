# Valtio - Proxy-Based State Management

## Overview

Valtio is a proxy-based state management library that makes state management in React simple and intuitive. It uses ES6 Proxies to automatically track state access and trigger re-renders only when necessary. The name "Valtio" means "state" in Finnish.

### Key Features

- **Proxy-based**: Leverages ES6 Proxies for automatic tracking
- **Mutable syntax**: Write code that looks like mutations
- **Minimal API**: Just wrap your state
- **Automatic optimization**: Only re-renders when accessed state changes
- **Nested objects**: Full support for deep nested structures
- **TypeScript support**: Excellent type inference
- **DevTools**: Redux DevTools integration

## Installation and Setup

```bash
# npm
npm install valtio

# yarn
yarn add valtio

# pnpm
pnpm add valtio
```

## Basic Usage

### Creating and Using Proxy State

```typescript
import { proxy, useSnapshot } from 'valtio';

// Create a proxy state
const state = proxy({
  count: 0,
  text: 'hello',
});

function Counter() {
  // useSnapshot creates a snapshot for tracking
  const snap = useSnapshot(state);

  return (
    <div>
      <p>Count: {snap.count}</p>
      {/* Direct mutation - looks like regular mutation but tracked */}
      <button onClick={() => state.count++}>Increment</button>
      <button onClick={() => state.count--}>Decrement</button>
      <button onClick={() => (state.count = 0)}>Reset</button>
    </div>
  );
}

function TextDisplay() {
  const snap = useSnapshot(state);
  
  return (
    <div>
      <p>Text: {snap.text}</p>
      <input
        value={snap.text}
        onChange={(e) => (state.text = e.target.value)}
      />
    </div>
  );
}
```

### Nested Objects

```typescript
import { proxy, useSnapshot } from 'valtio';

const store = proxy({
  user: {
    name: 'John Doe',
    email: 'john@example.com',
    address: {
      street: '123 Main St',
      city: 'New York',
      country: 'USA',
    },
  },
  preferences: {
    theme: 'light' as 'light' | 'dark',
    notifications: true,
  },
});

function UserProfile() {
  const snap = useSnapshot(store);

  return (
    <div>
      <h2>{snap.user.name}</h2>
      <p>{snap.user.email}</p>
      <p>
        {snap.user.address.city}, {snap.user.address.country}
      </p>
      
      <button onClick={() => {
        store.user.name = 'Jane Smith';
      }}>
        Update Name
      </button>
      
      <button onClick={() => {
        store.user.address.city = 'San Francisco';
      }}>
        Update City
      </button>
    </div>
  );
}

function ThemeToggle() {
  const snap = useSnapshot(store);

  return (
    <button onClick={() => {
      store.preferences.theme = 
        store.preferences.theme === 'light' ? 'dark' : 'light';
    }}>
      Current: {snap.preferences.theme}
    </button>
  );
}
```

## Arrays and Collections

### Managing Arrays

```typescript
import { proxy, useSnapshot } from 'valtio';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

const todoState = proxy({
  todos: [] as Todo[],
  filter: 'all' as 'all' | 'active' | 'completed',
});

function addTodo(text: string) {
  todoState.todos.push({
    id: crypto.randomUUID(),
    text,
    completed: false,
  });
}

function toggleTodo(id: string) {
  const todo = todoState.todos.find(t => t.id === id);
  if (todo) {
    todo.completed = !todo.completed;
  }
}

function deleteTodo(id: string) {
  const index = todoState.todos.findIndex(t => t.id === id);
  if (index !== -1) {
    todoState.todos.splice(index, 1);
  }
}

function TodoApp() {
  const snap = useSnapshot(todoState);
  const [newTodoText, setNewTodoText] = React.useState('');

  const filteredTodos = snap.todos.filter(todo => {
    if (snap.filter === 'active') return !todo.completed;
    if (snap.filter === 'completed') return todo.completed;
    return true;
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (newTodoText.trim()) {
      addTodo(newTodoText);
      setNewTodoText('');
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          value={newTodoText}
          onChange={(e) => setNewTodoText(e.target.value)}
          placeholder="Add todo..."
        />
        <button type="submit">Add</button>
      </form>

      <div>
        <button onClick={() => (todoState.filter = 'all')}>All</button>
        <button onClick={() => (todoState.filter = 'active')}>Active</button>
        <button onClick={() => (todoState.filter = 'completed')}>Completed</button>
      </div>

      <ul>
        {filteredTodos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Maps and Sets

```typescript
import { proxy, useSnapshot } from 'valtio';

const collectionState = proxy({
  userSet: new Set<string>(),
  userMap: new Map<string, { name: string; age: number }>(),
});

function addUser(id: string, name: string, age: number) {
  collectionState.userSet.add(id);
  collectionState.userMap.set(id, { name, age });
}

function removeUser(id: string) {
  collectionState.userSet.delete(id);
  collectionState.userMap.delete(id);
}

function UserList() {
  const snap = useSnapshot(collectionState);

  return (
    <div>
      <h3>Total Users: {snap.userSet.size}</h3>
      <ul>
        {Array.from(snap.userSet).map(id => {
          const user = snap.userMap.get(id);
          return (
            <li key={id}>
              {user?.name} ({user?.age})
              <button onClick={() => removeUser(id)}>Remove</button>
            </li>
          );
        })}
      </ul>
      <button onClick={() => addUser(
        crypto.randomUUID(),
        'New User',
        25
      )}>
        Add User
      </button>
    </div>
  );
}
```

## Computed Values with derive

```typescript
import { proxy, useSnapshot } from 'valtio';
import { derive } from 'valtio/utils';

const cartState = proxy({
  items: [] as Array<{ id: string; name: string; price: number; quantity: number }>,
  taxRate: 0.1,
});

// Derive computed values
const derivedCart = derive({
  subtotal: (get) => {
    const items = get(cartState).items;
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  },
  tax: (get) => {
    const subtotal = get(derivedCart).subtotal;
    const taxRate = get(cartState).taxRate;
    return subtotal * taxRate;
  },
  total: (get) => {
    const subtotal = get(derivedCart).subtotal;
    const tax = get(derivedCart).tax;
    return subtotal + tax;
  },
});

function addToCart(name: string, price: number) {
  const existingItem = cartState.items.find(item => item.name === name);
  if (existingItem) {
    existingItem.quantity++;
  } else {
    cartState.items.push({
      id: crypto.randomUUID(),
      name,
      price,
      quantity: 1,
    });
  }
}

function Cart() {
  const cartSnap = useSnapshot(cartState);
  const derivedSnap = useSnapshot(derivedCart);

  return (
    <div>
      <h2>Shopping Cart</h2>
      <ul>
        {cartSnap.items.map(item => (
          <li key={item.id}>
            {item.name} - ${item.price} x {item.quantity}
            <button onClick={() => {
              const foundItem = cartState.items.find(i => i.id === item.id);
              if (foundItem && foundItem.quantity > 1) {
                foundItem.quantity--;
              } else {
                const index = cartState.items.findIndex(i => i.id === item.id);
                cartState.items.splice(index, 1);
              }
            }}>
              -
            </button>
            <button onClick={() => {
              const foundItem = cartState.items.find(i => i.id === item.id);
              if (foundItem) foundItem.quantity++;
            }}>
              +
            </button>
          </li>
        ))}
      </ul>
      <div>
        <p>Subtotal: ${derivedSnap.subtotal.toFixed(2)}</p>
        <p>Tax: ${derivedSnap.tax.toFixed(2)}</p>
        <p><strong>Total: ${derivedSnap.total.toFixed(2)}</strong></p>
      </div>
      <button onClick={() => addToCart('Product', 29.99)}>Add Product</button>
    </div>
  );
}
```

## Subscriptions

### subscribe for Side Effects

```typescript
import { proxy, subscribe } from 'valtio';

const appState = proxy({
  notifications: [] as Array<{ id: string; message: string; type: 'info' | 'success' | 'error' }>,
  user: null as { name: string; email: string } | null,
});

// Subscribe to all changes
const unsubscribe = subscribe(appState, () => {
  console.log('State changed:', appState);
});

// Subscribe to specific property
subscribe(appState.user, () => {
  if (appState.user) {
    console.log('User changed:', appState.user);
    // Trigger side effect, e.g., save to localStorage
    localStorage.setItem('user', JSON.stringify(appState.user));
  }
});

// Subscribe with detailed operations
subscribe(appState.notifications, (ops) => {
  console.log('Operations:', ops);
  // ops contains details about what changed
});

function addNotification(message: string, type: 'info' | 'success' | 'error') {
  const notification = {
    id: crypto.randomUUID(),
    message,
    type,
  };
  
  appState.notifications.push(notification);
  
  // Auto-remove after 3 seconds
  setTimeout(() => {
    const index = appState.notifications.findIndex(n => n.id === notification.id);
    if (index !== -1) {
      appState.notifications.splice(index, 1);
    }
  }, 3000);
}

function NotificationList() {
  const snap = useSnapshot(appState);

  return (
    <div>
      {snap.notifications.map(notif => (
        <div key={notif.id} className={`notification ${notif.type}`}>
          {notif.message}
        </div>
      ))}
    </div>
  );
}
```

## Async Operations

### Handling Async State

```typescript
import { proxy, useSnapshot } from 'valtio';

interface Product {
  id: string;
  name: string;
  price: number;
}

const productState = proxy({
  products: [] as Product[],
  loading: false,
  error: null as string | null,
});

async function fetchProducts() {
  productState.loading = true;
  productState.error = null;
  
  try {
    const response = await fetch('/api/products');
    if (!response.ok) throw new Error('Failed to fetch products');
    const data = await response.json();
    productState.products = data;
  } catch (error) {
    productState.error = (error as Error).message;
  } finally {
    productState.loading = false;
  }
}

async function addProduct(product: Omit<Product, 'id'>) {
  const optimisticId = `temp-${Date.now()}`;
  const optimisticProduct = { ...product, id: optimisticId };
  
  // Optimistic update
  productState.products.push(optimisticProduct);
  
  try {
    const response = await fetch('/api/products', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(product),
    });
    
    const savedProduct = await response.json();
    
    // Replace optimistic product with real one
    const index = productState.products.findIndex(p => p.id === optimisticId);
    if (index !== -1) {
      productState.products[index] = savedProduct;
    }
  } catch (error) {
    // Remove optimistic product on error
    const index = productState.products.findIndex(p => p.id === optimisticId);
    if (index !== -1) {
      productState.products.splice(index, 1);
    }
    productState.error = (error as Error).message;
  }
}

function ProductList() {
  const snap = useSnapshot(productState);

  React.useEffect(() => {
    fetchProducts();
  }, []);

  if (snap.loading) return <div>Loading...</div>;
  if (snap.error) return <div>Error: {snap.error}</div>;

  return (
    <div>
      <ul>
        {snap.products.map(product => (
          <li key={product.id}>
            {product.name} - ${product.price}
          </li>
        ))}
      </ul>
      <button onClick={() => addProduct({ name: 'New Product', price: 99.99 })}>
        Add Product
      </button>
    </div>
  );
}
```

## Ref and Watch Utilities

```typescript
import { proxy, useSnapshot, ref } from 'valtio';
import { watch } from 'valtio/utils';

// ref() marks objects that shouldn't be proxied
const mediaState = proxy({
  videoElement: ref(null as HTMLVideoElement | null),
  currentTime: 0,
  duration: 0,
  playing: false,
});

// Watch specific values
watch((get) => {
  const playing = get(mediaState).playing;
  const video = get(mediaState).videoElement;
  
  if (video) {
    if (playing) {
      video.play();
    } else {
      video.pause();
    }
  }
});

function VideoPlayer() {
  const snap = useSnapshot(mediaState);
  const videoRef = React.useRef<HTMLVideoElement>(null);

  React.useEffect(() => {
    if (videoRef.current) {
      mediaState.videoElement = ref(videoRef.current);
      
      const video = videoRef.current;
      
      const updateTime = () => {
        mediaState.currentTime = video.currentTime;
      };
      
      const updateDuration = () => {
        mediaState.duration = video.duration;
      };
      
      video.addEventListener('timeupdate', updateTime);
      video.addEventListener('loadedmetadata', updateDuration);
      
      return () => {
        video.removeEventListener('timeupdate', updateTime);
        video.removeEventListener('loadedmetadata', updateDuration);
      };
    }
  }, []);

  return (
    <div>
      <video ref={videoRef} src="/video.mp4" />
      <div>
        <button onClick={() => (mediaState.playing = !mediaState.playing)}>
          {snap.playing ? 'Pause' : 'Play'}
        </button>
        <p>
          {snap.currentTime.toFixed(1)}s / {snap.duration.toFixed(1)}s
        </p>
      </div>
    </div>
  );
}
```

## DevTools Integration

```typescript
import { proxy, useSnapshot } from 'valtio';
import { devtools } from 'valtio/utils';

const store = proxy({
  count: 0,
  text: 'hello',
});

// Enable Redux DevTools
devtools(store, { name: 'App Store', enabled: true });

function Counter() {
  const snap = useSnapshot(store);

  return (
    <div>
      <p>Count: {snap.count}</p>
      <button onClick={() => store.count++}>Increment</button>
      <p>Text: {snap.text}</p>
      <input
        value={snap.text}
        onChange={(e) => (store.text = e.target.value)}
      />
    </div>
  );
}
```

## Organizing State

### Modular State Pattern

```typescript
import { proxy, useSnapshot } from 'valtio';

// User module
const userModule = proxy({
  currentUser: null as { id: string; name: string; email: string } | null,
  isAuthenticated: false,
  
  login(user: { id: string; name: string; email: string }) {
    this.currentUser = user;
    this.isAuthenticated = true;
  },
  
  logout() {
    this.currentUser = null;
    this.isAuthenticated = false;
  },
});

// Cart module
const cartModule = proxy({
  items: [] as Array<{ id: string; name: string; price: number; quantity: number }>,
  
  addItem(item: { id: string; name: string; price: number }) {
    const existing = this.items.find(i => i.id === item.id);
    if (existing) {
      existing.quantity++;
    } else {
      this.items.push({ ...item, quantity: 1 });
    }
  },
  
  removeItem(id: string) {
    const index = this.items.findIndex(i => i.id === id);
    if (index !== -1) {
      this.items.splice(index, 1);
    }
  },
  
  clear() {
    this.items = [];
  },
});

// Root store combining modules
const store = proxy({
  user: userModule,
  cart: cartModule,
});

export { store, userModule, cartModule };

function App() {
  const snap = useSnapshot(store);

  return (
    <div>
      {snap.user.isAuthenticated ? (
        <>
          <p>Welcome, {snap.user.currentUser?.name}!</p>
          <button onClick={() => store.user.logout()}>Logout</button>
          <p>Cart items: {snap.cart.items.length}</p>
        </>
      ) : (
        <button onClick={() => store.user.login({
          id: '1',
          name: 'John Doe',
          email: 'john@example.com'
        })}>
          Login
        </button>
      )}
    </div>
  );
}
```

## Testing

```typescript
import { proxy, snapshot } from 'valtio';

const testState = proxy({
  count: 0,
  increment() {
    this.count++;
  },
  decrement() {
    this.count--;
  },
});

describe('valtio state', () => {
  beforeEach(() => {
    testState.count = 0;
  });

  it('should increment count', () => {
    testState.increment();
    expect(snapshot(testState).count).toBe(1);
  });

  it('should decrement count', () => {
    testState.decrement();
    expect(snapshot(testState).count).toBe(-1);
  });

  it('should handle multiple operations', () => {
    testState.increment();
    testState.increment();
    testState.decrement();
    expect(snapshot(testState).count).toBe(1);
  });
});
```

## Common Mistakes

1. **Forgetting useSnapshot**: Must use useSnapshot to create reactive snapshots
2. **Mutating snapshot**: Never mutate snapshot, mutate original proxy state
3. **Over-proxying**: Use ref() for objects that shouldn't be proxied (DOM elements, class instances)
4. **Snapshot in callbacks**: Don't capture snapshots in callbacks, use state directly
5. **Not handling async properly**: Track loading and error states

## Best Practices

1. **Use useSnapshot in components**: Always create snapshots for React tracking
2. **Mutate state directly**: Write natural mutation code
3. **Organize with modules**: Split large states into focused modules
4. **Use ref() appropriately**: For non-serializable objects
5. **Add methods to state**: Co-locate actions with state
6. **Subscribe for effects**: Use subscribe for side effects outside React
7. **Use derive for computed**: Create derived computed values efficiently

## When to Use Valtio

### Good Use Cases
- Applications with complex nested state
- Teams preferring mutable syntax
- Rapid prototyping
- State with many nested objects
- When you want minimal boilerplate

### Not Ideal For
- Immutability requirements
- Redux DevTools time-travel needed
- Environments without Proxy support

## Interview Questions

1. **How does Valtio track state changes?**
   - Uses ES6 Proxies to automatically track access and mutations

2. **What is the difference between state and snapshot?**
   - State is the mutable proxy; snapshot is the immutable read-only view

3. **When should you use ref()?**
   - For objects that shouldn't be proxied, like DOM elements or class instances

4. **How do you create computed values?**
   - Use derive() utility or create getters in the proxy

5. **Can you use Valtio outside React?**
   - Yes, using snapshot() and subscribe()

## Key Takeaways

- Valtio uses Proxy for automatic state tracking
- Write mutable-looking code that's actually tracked
- useSnapshot creates reactive snapshots for components
- Full support for nested objects and collections
- ref() for non-proxied objects
- derive() for computed values
- subscribe() for side effects
- Natural JavaScript syntax with automatic optimization

## Resources

- [Official Documentation](https://github.com/pmndrs/valtio)
- [API Reference](https://github.com/pmndrs/valtio#api)
- [Utilities](https://github.com/pmndrs/valtio#utilities)
- [Examples](https://github.com/pmndrs/valtio/tree/main/examples)
