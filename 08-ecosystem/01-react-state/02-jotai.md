# Jotai - Primitive and Flexible State Management

## The Idea

**In plain English:** Jotai is a tool for React apps that lets you store and share pieces of information (called "atoms") across your whole app, so any part of the page can read or update that information without having to pass it around manually. An "atom" here just means a single, small container that holds one piece of data.

**Real-world analogy:** Imagine a school notice board where teachers pin individual cards — one card for the lunch menu, one for tomorrow's weather, one for upcoming events. Any student or teacher in the school can walk up, read any card, or swap it out with an updated one, without needing to go through a central office.

- The notice board = the Jotai store (the shared space that holds all the data)
- Each pinned card = an atom (a small, independent container for one piece of information)
- A student reading a card = a component using `useAtomValue` to read state
- A teacher replacing a card = a component using `useSetAtom` to update state

---

## Overview

Jotai is a primitive and flexible state management library for React. It takes an atomic approach to global React state management with a bottom-up model inspired by Recoil. The name "Jotai" means "state" in Japanese.

### Key Features

- **Atomic**: State is broken down into atoms
- **TypeScript-oriented**: First-class TypeScript support
- **Minimal core API**: Simple and intuitive
- **No Context Provider**: Works out of the box
- **Async support**: Built-in async atom support
- **DevTools integration**: Debugging capabilities
- **Framework agnostic**: Can be used with Next.js, Gatsby, etc.

## Installation and Setup

```bash
# npm
npm install jotai

# yarn
yarn add jotai

# pnpm
pnpm add jotai
```

## Basic Atoms

### Primitive Atoms

```typescript
import { atom, useAtom } from 'jotai';

// Create primitive atoms
const countAtom = atom(0);
const nameAtom = atom('John Doe');
const isDarkModeAtom = atom(false);

// Using atoms in components
function Counter() {
  const [count, setCount] = useAtom(countAtom);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

function UserName() {
  const [name, setName] = useAtom(nameAtom);

  return (
    <div>
      <p>Name: {name}</p>
      <input 
        value={name} 
        onChange={(e) => setName(e.target.value)} 
      />
    </div>
  );
}

// Read-only usage
function DisplayCount() {
  const [count] = useAtom(countAtom);
  return <p>Current count: {count}</p>;
}
```

### Read-Only Atoms with useAtomValue

```typescript
import { atom, useAtomValue, useSetAtom } from 'jotai';

const temperatureAtom = atom(20);

// More semantic read-only access
function TemperatureDisplay() {
  const temperature = useAtomValue(temperatureAtom);
  return <div>Temperature: {temperature}°C</div>;
}

// Write-only access
function TemperatureControls() {
  const setTemperature = useSetAtom(temperatureAtom);
  
  return (
    <div>
      <button onClick={() => setTemperature((t) => t + 1)}>Increase</button>
      <button onClick={() => setTemperature((t) => t - 1)}>Decrease</button>
    </div>
  );
}
```

## Derived Atoms

### Read-Only Derived Atoms

```typescript
import { atom, useAtomValue } from 'jotai';

const firstNameAtom = atom('John');
const lastNameAtom = atom('Doe');

// Derived atom that reads from other atoms
const fullNameAtom = atom((get) => {
  const firstName = get(firstNameAtom);
  const lastName = get(lastNameAtom);
  return `${firstName} ${lastName}`;
});

function FullNameDisplay() {
  const fullName = useAtomValue(fullNameAtom);
  return <div>Full Name: {fullName}</div>;
}

// More complex derived atom
const priceAtom = atom(100);
const quantityAtom = atom(1);
const taxRateAtom = atom(0.1);

const totalPriceAtom = atom((get) => {
  const price = get(priceAtom);
  const quantity = get(quantityAtom);
  const taxRate = get(taxRateAtom);
  const subtotal = price * quantity;
  const tax = subtotal * taxRate;
  return subtotal + tax;
});

function ShoppingCart() {
  const totalPrice = useAtomValue(totalPriceAtom);
  const [quantity, setQuantity] = useAtom(quantityAtom);
  
  return (
    <div>
      <p>Quantity: {quantity}</p>
      <button onClick={() => setQuantity(q => q + 1)}>Add</button>
      <p>Total: ${totalPrice.toFixed(2)}</p>
    </div>
  );
}
```

### Writable Derived Atoms

```typescript
import { atom, useAtom } from 'jotai';

const celsiusAtom = atom(0);

// Derived atom with read and write functions
const fahrenheitAtom = atom(
  (get) => get(celsiusAtom) * 9/5 + 32,
  (get, set, newFahrenheit: number) => {
    set(celsiusAtom, (newFahrenheit - 32) * 5/9);
  }
);

function TemperatureConverter() {
  const [celsius, setCelsius] = useAtom(celsiusAtom);
  const [fahrenheit, setFahrenheit] = useAtom(fahrenheitAtom);

  return (
    <div>
      <div>
        <label>Celsius: </label>
        <input 
          type="number"
          value={celsius} 
          onChange={(e) => setCelsius(Number(e.target.value))} 
        />
      </div>
      <div>
        <label>Fahrenheit: </label>
        <input 
          type="number"
          value={fahrenheit} 
          onChange={(e) => setFahrenheit(Number(e.target.value))} 
        />
      </div>
    </div>
  );
}
```

### Complex Writable Derived Atoms

```typescript
import { atom, useAtom } from 'jotai';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

const todosAtom = atom<Todo[]>([]);

// Derived atom for filtering todos
const filterAtom = atom<'all' | 'active' | 'completed'>('all');

const filteredTodosAtom = atom(
  (get) => {
    const todos = get(todosAtom);
    const filter = get(filterAtom);
    
    if (filter === 'active') return todos.filter(t => !t.completed);
    if (filter === 'completed') return todos.filter(t => t.completed);
    return todos;
  }
);

// Action atoms
const addTodoAtom = atom(
  null,
  (get, set, text: string) => {
    const newTodo: Todo = {
      id: crypto.randomUUID(),
      text,
      completed: false,
    };
    set(todosAtom, [...get(todosAtom), newTodo]);
  }
);

const toggleTodoAtom = atom(
  null,
  (get, set, id: string) => {
    set(todosAtom, get(todosAtom).map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  }
);

const deleteTodoAtom = atom(
  null,
  (get, set, id: string) => {
    set(todosAtom, get(todosAtom).filter(todo => todo.id !== id));
  }
);

function TodoApp() {
  const [filter, setFilter] = useAtom(filterAtom);
  const filteredTodos = useAtomValue(filteredTodosAtom);
  const addTodo = useSetAtom(addTodoAtom);
  const toggleTodo = useSetAtom(toggleTodoAtom);
  const deleteTodo = useSetAtom(deleteTodoAtom);
  
  const [newTodoText, setNewTodoText] = React.useState('');

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
        <button onClick={() => setFilter('all')}>All</button>
        <button onClick={() => setFilter('active')}>Active</button>
        <button onClick={() => setFilter('completed')}>Completed</button>
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

## Async Atoms

### Basic Async Atoms

```typescript
import { atom, useAtomValue } from 'jotai';

interface User {
  id: string;
  name: string;
  email: string;
}

// Async atom that fetches data
const userAtom = atom(async () => {
  const response = await fetch('/api/user');
  if (!response.ok) throw new Error('Failed to fetch user');
  return response.json() as Promise<User>;
});

function UserProfile() {
  const user = useAtomValue(userAtom);
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// Wrap in Suspense
function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserProfile />
    </Suspense>
  );
}
```

### Async Atoms with Dependencies

```typescript
import { atom, useAtom, useAtomValue } from 'jotai';

const userIdAtom = atom<string | null>(null);

// Async atom that depends on another atom
const userDataAtom = atom(async (get) => {
  const userId = get(userIdAtom);
  if (!userId) return null;
  
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) throw new Error('Failed to fetch user');
  return response.json();
});

function UserSelector() {
  const [userId, setUserId] = useAtom(userIdAtom);
  
  return (
    <select value={userId || ''} onChange={(e) => setUserId(e.target.value)}>
      <option value="">Select user...</option>
      <option value="1">User 1</option>
      <option value="2">User 2</option>
    </select>
  );
}

function UserDetails() {
  const userData = useAtomValue(userDataAtom);
  
  if (!userData) return <div>Select a user</div>;
  
  return (
    <div>
      <h3>{userData.name}</h3>
      <p>{userData.email}</p>
    </div>
  );
}

function App() {
  return (
    <div>
      <UserSelector />
      <Suspense fallback={<div>Loading user...</div>}>
        <UserDetails />
      </Suspense>
    </div>
  );
}
```

### Async Writable Atoms

```typescript
import { atom, useAtom } from 'jotai';

interface Product {
  id: string;
  name: string;
  price: number;
}

const productsAtom = atom<Product[]>([]);

// Async atom for fetching products
const fetchProductsAtom = atom(
  async (get) => {
    const response = await fetch('/api/products');
    return response.json() as Promise<Product[]>;
  },
  async (get, set) => {
    const products = await get(fetchProductsAtom);
    set(productsAtom, products);
  }
);

// Async atom for adding a product
const addProductAtom = atom(
  null,
  async (get, set, product: Omit<Product, 'id'>) => {
    const response = await fetch('/api/products', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(product),
    });
    
    if (!response.ok) throw new Error('Failed to add product');
    
    const newProduct = await response.json();
    set(productsAtom, [...get(productsAtom), newProduct]);
  }
);

function ProductList() {
  const [products, refetch] = useAtom(fetchProductsAtom);
  const addProduct = useSetAtom(addProductAtom);

  const handleAdd = async () => {
    await addProduct({ name: 'New Product', price: 99.99 });
    refetch();
  };

  return (
    <div>
      <button onClick={handleAdd}>Add Product</button>
      <ul>
        {products.map(product => (
          <li key={product.id}>{product.name} - ${product.price}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Atom Families

### atomFamily for Dynamic Atoms

```typescript
import { atomFamily, useAtomValue } from 'jotai/utils';

interface User {
  id: string;
  name: string;
  email: string;
}

// Create atoms dynamically based on parameters
const userAtomFamily = atomFamily((userId: string) =>
  atom(async () => {
    const response = await fetch(`/api/users/${userId}`);
    return response.json() as Promise<User>;
  })
);

function UserCard({ userId }: { userId: string }) {
  const user = useAtomValue(userAtomFamily(userId));
  
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}

function UserList() {
  const userIds = ['1', '2', '3'];
  
  return (
    <div>
      {userIds.map(userId => (
        <Suspense key={userId} fallback={<div>Loading...</div>}>
          <UserCard userId={userId} />
        </Suspense>
      ))}
    </div>
  );
}
```

### atomFamily with Complex Keys

```typescript
import { atomFamily } from 'jotai/utils';

interface QueryParams {
  category: string;
  page: number;
  limit: number;
}

// Use JSON.stringify for complex keys
const queryAtomFamily = atomFamily((params: QueryParams) => {
  return atom(async () => {
    const queryString = new URLSearchParams({
      category: params.category,
      page: String(params.page),
      limit: String(params.limit),
    }).toString();
    
    const response = await fetch(`/api/items?${queryString}`);
    return response.json();
  });
}, (a, b) => JSON.stringify(a) === JSON.stringify(b));

function ProductGrid() {
  const params = { category: 'electronics', page: 1, limit: 20 };
  const products = useAtomValue(queryAtomFamily(params));
  
  return (
    <div>
      {products.map((product: any) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

## Storage Utilities

### atomWithStorage

```typescript
import { atomWithStorage } from 'jotai/utils';

// Persists to localStorage automatically
const darkModeAtom = atomWithStorage('darkMode', false);
const userPreferencesAtom = atomWithStorage('userPreferences', {
  language: 'en',
  notifications: true,
  theme: 'light',
});

function Settings() {
  const [darkMode, setDarkMode] = useAtom(darkModeAtom);
  const [preferences, setPreferences] = useAtom(userPreferencesAtom);

  return (
    <div>
      <label>
        <input
          type="checkbox"
          checked={darkMode}
          onChange={(e) => setDarkMode(e.target.checked)}
        />
        Dark Mode
      </label>
      
      <select
        value={preferences.language}
        onChange={(e) => setPreferences({
          ...preferences,
          language: e.target.value
        })}
      >
        <option value="en">English</option>
        <option value="es">Spanish</option>
        <option value="fr">French</option>
      </select>
    </div>
  );
}
```

### Custom Storage Implementation

```typescript
import { atomWithStorage, createJSONStorage } from 'jotai/utils';

// Custom storage adapter
const sessionStorageAdapter = createJSONStorage<any>(() => sessionStorage);

const sessionDataAtom = atomWithStorage(
  'sessionData',
  { lastVisit: new Date().toISOString() },
  sessionStorageAdapter
);

// IndexedDB storage (simplified)
const indexedDBStorage = {
  getItem: async (key: string): Promise<string | null> => {
    // IndexedDB get implementation
    return localStorage.getItem(key); // Fallback for example
  },
  setItem: async (key: string, value: string): Promise<void> => {
    // IndexedDB set implementation
    localStorage.setItem(key, value); // Fallback for example
  },
  removeItem: async (key: string): Promise<void> => {
    // IndexedDB remove implementation
    localStorage.removeItem(key); // Fallback for example
  },
};

const persistentDataAtom = atomWithStorage(
  'persistentData',
  { version: 1 },
  indexedDBStorage
);
```

## Atom Effects and Utilities

### atomWithReset

```typescript
import { atomWithReset, useResetAtom } from 'jotai/utils';

const formDataAtom = atomWithReset({
  name: '',
  email: '',
  message: '',
});

function ContactForm() {
  const [formData, setFormData] = useAtom(formDataAtom);
  const resetForm = useResetAtom(formDataAtom);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    await fetch('/api/contact', {
      method: 'POST',
      body: JSON.stringify(formData),
    });
    resetForm(); // Reset to initial values
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
      />
      <input
        value={formData.email}
        onChange={(e) => setFormData({ ...formData, email: e.target.value })}
      />
      <textarea
        value={formData.message}
        onChange={(e) => setFormData({ ...formData, message: e.target.value })}
      />
      <button type="submit">Submit</button>
      <button type="button" onClick={resetForm}>Reset</button>
    </form>
  );
}
```

### atomWithReducer

```typescript
import { atomWithReducer } from 'jotai/utils';

interface State {
  count: number;
  history: number[];
}

type Action = 
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return {
        count: state.count + 1,
        history: [...state.history, state.count + 1],
      };
    case 'decrement':
      return {
        count: state.count - 1,
        history: [...state.history, state.count - 1],
      };
    case 'reset':
      return { count: 0, history: [0] };
    default:
      return state;
  }
}

const counterAtom = atomWithReducer({ count: 0, history: [0] }, reducer);

function CounterWithHistory() {
  const [state, dispatch] = useAtom(counterAtom);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
      <p>History: {state.history.join(', ')}</p>
    </div>
  );
}
```

## Scoped Atoms with Provider

```typescript
import { Provider, atom, useAtom } from 'jotai';

const scopedCountAtom = atom(0);

function Counter() {
  const [count, setCount] = useAtom(scopedCountAtom);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}

function App() {
  return (
    <div>
      <h2>Counter 1 (Scoped)</h2>
      <Provider>
        <Counter />
      </Provider>
      
      <h2>Counter 2 (Scoped)</h2>
      <Provider>
        <Counter />
      </Provider>
      
      <h2>Counter 3 (Global)</h2>
      <Counter />
    </div>
  );
}
```

## DevTools Integration

```typescript
import { useAtom } from 'jotai';
import { useAtomDevtools } from 'jotai-devtools';

const countAtom = atom(0);
countAtom.debugLabel = 'count';

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  useAtomDevtools(countAtom);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}

// Or use the DevTools component
import { DevTools } from 'jotai-devtools';

function App() {
  return (
    <>
      <DevTools />
      <Counter />
    </>
  );
}
```

## Common Mistakes

1. **Not using Suspense with async atoms**: Always wrap in Suspense boundary
2. **Creating atoms inside components**: Create atoms at module level
3. **Over-using atom families**: Only use when truly dynamic
4. **Not labeling atoms for debugging**: Use debugLabel in development
5. **Forgetting cleanup in effects**: Return cleanup functions

## Best Practices

1. **Use TypeScript**: Leverage full type inference
2. **Keep atoms focused**: Single responsibility principle
3. **Use derived atoms**: Compute values instead of duplicating state
4. **Leverage Suspense**: For async operations and loading states
5. **Use atom families**: For dynamic, parameterized state
6. **Debug labels**: Always add debugLabel for easier debugging
7. **Split read/write**: Use useAtomValue and useSetAtom when appropriate

## When to Use Jotai

### Good Use Cases

- Bottom-up state management
- TypeScript projects
- Async state handling
- Derived state computations
- Applications requiring fine-grained updates

### Not Ideal For

- Simple local state (use useState)
- Redux DevTools time-travel features
- Very small projects

## Interview Questions

1. **What is the difference between atom and derived atom?**
   - Atom holds primitive state; derived atom computes from other atoms

2. **How does Jotai handle async state?**
   - Built-in async support with Suspense and error boundaries

3. **What are atom families used for?**
   - Creating dynamic atoms based on parameters

4. **How do you persist atoms?**
   - Use atomWithStorage utility

5. **Can you use Jotai without React?**
   - Yes, with jotai/vanilla for non-React usage

## Key Takeaways

- Jotai provides atomic, bottom-up state management
- Minimal API with powerful composition
- Built-in async support with Suspense
- Atom families enable dynamic state creation
- Storage utilities for persistence
- Full TypeScript support with excellent inference
- DevTools integration for debugging
- No Provider wrapping required (unless scoping needed)

## Resources

- [Official Documentation](https://jotai.org/)
- [GitHub Repository](https://github.com/pmndrs/jotai)
- [API Reference](https://jotai.org/docs/api/core)
- [Utilities](https://jotai.org/docs/utilities/storage)
- [DevTools](https://jotai.org/docs/tools/devtools)
