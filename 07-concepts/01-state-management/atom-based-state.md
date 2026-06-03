# Atom-Based State Management

## Overview

Atom-based state management represents a paradigm shift from traditional centralized stores to a distributed, atomic model where state is broken into small, independent units called "atoms." This approach, popularized by libraries like Recoil and Jotai, provides fine-grained reactivity, better performance through granular subscriptions, and a more flexible mental model for managing application state. Unlike Redux's single store or Context's provider-based approach, atom-based state enables components to subscribe to only the specific pieces of state they need.

## The Atom Concept

An atom is the smallest unit of state that can be independently managed and subscribed to:

```
Traditional Redux Store:
┌────────────────────────────────────┐
│         Global Store               │
│  ┌──────────────────────────────┐ │
│  │ users, todos, cart, theme,   │ │
│  │ notifications, settings, ... │ │
│  └──────────────────────────────┘ │
└────────────────────────────────────┘
      │
      └──→ Component subscribes to entire store
           (or large slices)

Atom-Based State:
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│User │  │Todos│  │Cart │  │Theme│  │ ... │
│Atom │  │Atom │  │Atom │  │Atom │  │     │
└──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘  └─────┘
   │        │        │        │
   └────────┴────────┴────────┴──────→ Components subscribe
                                        to individual atoms
```

## Core Principles

### 1. Atomic State Units

Each piece of state is independent and self-contained:

```typescript
// Each atom is independent
const userAtom = atom({ name: 'John' });
const themeAtom = atom('dark');
const countAtom = atom(0);

// Components subscribe to only what they need
function UserProfile() {
  const user = useAtom(userAtom); // Only re-renders when user changes
}

function ThemeToggle() {
  const theme = useAtom(themeAtom); // Only re-renders when theme changes
}
```

### 2. Granular Subscriptions

Components only re-render when atoms they use change:

```
Component Subscriptions:

UserProfile     →  [userAtom]
ThemeToggle     →  [themeAtom]
TodoList        →  [todosAtom, filterAtom]
ShoppingCart    →  [cartAtom, totalAtom]

When userAtom changes:
✓ UserProfile re-renders
✗ ThemeToggle doesn't re-render
✗ TodoList doesn't re-render
✗ ShoppingCart doesn't re-render
```

### 3. Derived State as Atoms

Computed values are first-class atoms:

```typescript
// Base atoms
const todosAtom = atom([...]);
const filterAtom = atom('all');

// Derived atom (selector)
const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(filterAtom);
  
  switch (filter) {
    case 'active': return todos.filter(t => !t.completed);
    case 'completed': return todos.filter(t => t.completed);
    default: return todos;
  }
});

// Component uses derived atom like any other atom
function TodoList() {
  const filteredTodos = useAtom(filteredTodosAtom);
}
```

## Jotai: Primitive Atomic State

Jotai (Japanese for "state") is a minimal atomic state library:

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// 1. Create atoms
const countAtom = atom(0);

// Atom with default value
const userAtom = atom<User | null>(null);

// Atom with initialization function
const timestampAtom = atom(() => Date.now());

// 2. Read and write atom
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(c => c + 1)}>Increment (updater)</button>
    </div>
  );
}

// 3. Read-only hook (performance optimization)
function DisplayCount() {
  const count = useAtomValue(countAtom);
  // Component doesn't have setCount, signals it won't update
  return <div>Count is: {count}</div>;
}

// 4. Write-only hook
function ResetButton() {
  const setCount = useSetAtom(countAtom);
  // Component only writes, won't re-render when count changes
  return <button onClick={() => setCount(0)}>Reset</button>;
}
```

### Jotai Derived Atoms (Read-Only)

```typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Base atoms
const todosAtom = atom<Todo[]>([]);
const filterAtom = atom<'all' | 'active' | 'completed'>('all');
const searchAtom = atom('');

// Derived atom - automatically recomputes when dependencies change
const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(filterAtom);
  const search = get(searchAtom);
  
  let filtered = todos;
  
  // Apply filter
  if (filter === 'active') {
    filtered = filtered.filter(t => !t.completed);
  } else if (filter === 'completed') {
    filtered = filtered.filter(t => t.completed);
  }
  
  // Apply search
  if (search) {
    filtered = filtered.filter(t =>
      t.text.toLowerCase().includes(search.toLowerCase())
    );
  }
  
  return filtered;
});

// Statistics derived atom
const todoStatsAtom = atom((get) => {
  const todos = get(todosAtom);
  return {
    total: todos.length,
    completed: todos.filter(t => t.completed).length,
    active: todos.filter(t => !t.completed).length
  };
});

// Usage
function TodoList() {
  const todos = useAtomValue(filteredTodosAtom);
  const stats = useAtomValue(todoStatsAtom);
  
  return (
    <div>
      <p>Showing {todos.length} of {stats.total} todos</p>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Jotai Writable Derived Atoms

```typescript
// Read-write derived atom
const doubleCountAtom = atom(
  // Read function
  (get) => get(countAtom) * 2,
  
  // Write function
  (get, set, newValue: number) => {
    set(countAtom, newValue / 2);
  }
);

function DoubleCounter() {
  const [double, setDouble] = useAtom(doubleCountAtom);
  
  return (
    <div>
      <p>Double: {double}</p>
      <button onClick={() => setDouble(double + 2)}>Add 2</button>
      {/* Updates countAtom by 1 */}
    </div>
  );
}

// More complex example: Celsius/Fahrenheit
const celsiusAtom = atom(0);

const fahrenheitAtom = atom(
  (get) => (get(celsiusAtom) * 9) / 5 + 32,
  (get, set, newValue: number) => {
    set(celsiusAtom, ((newValue - 32) * 5) / 9);
  }
);

function TemperatureConverter() {
  const [celsius, setCelsius] = useAtom(celsiusAtom);
  const [fahrenheit, setFahrenheit] = useAtom(fahrenheitAtom);
  
  return (
    <div>
      <input
        type="number"
        value={celsius}
        onChange={e => setCelsius(Number(e.target.value))}
      />°C
      
      <input
        type="number"
        value={fahrenheit}
        onChange={e => setFahrenheit(Number(e.target.value))}
      />°F
    </div>
  );
}
```

### Jotai Async Atoms

```typescript
// Async atom
const userIdAtom = atom<string | null>(null);

const userAtom = atom(async (get) => {
  const userId = get(userIdAtom);
  if (!userId) return null;
  
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
});

// Usage with Suspense
function UserProfile() {
  const user = useAtomValue(userAtom);
  
  if (!user) return <div>No user selected</div>;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

function App() {
  return (
    <Suspense fallback={<div>Loading user...</div>}>
      <UserProfile />
    </Suspense>
  );
}

// Async atom with dependencies
const todoIdsAtom = atom<string[]>([]);

const todosAtom = atom(async (get) => {
  const ids = get(todoIdsAtom);
  
  const todos = await Promise.all(
    ids.map(id => fetch(`/api/todos/${id}`).then(r => r.json()))
  );
  
  return todos;
});

// Writable async atom
const saveTodoAtom = atom(
  null, // No read function
  async (get, set, todo: Todo) => {
    // Optimistically update
    set(todosAtom, (prev) => [...prev, todo]);
    
    try {
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(todo)
      });
      const savedTodo = await response.json();
      
      // Update with server response
      set(todosAtom, (prev) =>
        prev.map(t => (t.id === todo.id ? savedTodo : t))
      );
    } catch (error) {
      // Rollback on error
      set(todosAtom, (prev) => prev.filter(t => t.id !== todo.id));
      throw error;
    }
  }
);
```

### Jotai Atom Families

Create atoms dynamically with parameters:

```typescript
import { atomFamily } from 'jotai/utils';

// Atom family for todo items
const todoAtomFamily = atomFamily((id: string) =>
  atom(async () => {
    const response = await fetch(`/api/todos/${id}`);
    return response.json();
  })
);

// Usage - creates separate atom for each ID
function TodoItem({ id }: { id: string }) {
  const todo = useAtomValue(todoAtomFamily(id));
  
  return <div>{todo.text}</div>;
}

// Atom family with primitive values
const counterFamily = atomFamily((id: string) => atom(0));

function Counter({ id }: { id: string }) {
  const [count, setCount] = useAtom(counterFamily(id));
  
  return (
    <div>
      Counter {id}: {count}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// Multiple counters with independent state
<Counter id="a" /> {/* Has its own count */}
<Counter id="b" /> {/* Has separate count */}
```

## Recoil: Facebook's Atomic State

Recoil is Facebook's atomic state library with more features than Jotai:

```typescript
import { atom, selector, useRecoilState, useRecoilValue, useSetRecoilState } from 'recoil';

// 1. Create atom with unique key
const countState = atom({
  key: 'countState', // Unique ID for this atom
  default: 0
});

const userState = atom<User | null>({
  key: 'userState',
  default: null
});

// 2. Use in components
function Counter() {
  const [count, setCount] = useRecoilState(countState);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// 3. Read-only
function DisplayCount() {
  const count = useRecoilValue(countState);
  return <div>Count is: {count}</div>;
}

// 4. Write-only
function ResetButton() {
  const setCount = useSetRecoilState(countState);
  return <button onClick={() => setCount(0)}>Reset</button>;
}

// 5. Wrap app in RecoilRoot
import { RecoilRoot } from 'recoil';

function App() {
  return (
    <RecoilRoot>
      <Counter />
      <DisplayCount />
      <ResetButton />
    </RecoilRoot>
  );
}
```

### Recoil Selectors (Derived State)

```typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Base atoms
const todoListState = atom<Todo[]>({
  key: 'todoListState',
  default: []
});

const todoListFilterState = atom<'all' | 'active' | 'completed'>({
  key: 'todoListFilterState',
  default: 'all'
});

// Derived state with selector
const filteredTodoListState = selector({
  key: 'filteredTodoListState',
  get: ({ get }) => {
    const filter = get(todoListFilterState);
    const list = get(todoListState);
    
    switch (filter) {
      case 'active':
        return list.filter(todo => !todo.completed);
      case 'completed':
        return list.filter(todo => todo.completed);
      default:
        return list;
    }
  }
});

// Statistics selector
const todoListStatsState = selector({
  key: 'todoListStatsState',
  get: ({ get }) => {
    const todoList = get(todoListState);
    const totalNum = todoList.length;
    const totalCompletedNum = todoList.filter(t => t.completed).length;
    const totalUncompletedNum = totalNum - totalCompletedNum;
    const percentCompleted = totalNum === 0 ? 0 : (totalCompletedNum / totalNum) * 100;
    
    return {
      totalNum,
      totalCompletedNum,
      totalUncompletedNum,
      percentCompleted
    };
  }
});

// Usage
function TodoList() {
  const todos = useRecoilValue(filteredTodoListState);
  const stats = useRecoilValue(todoListStatsState);
  
  return (
    <div>
      <h2>
        {stats.totalCompletedNum} / {stats.totalNum} completed 
        ({stats.percentCompleted.toFixed(0)}%)
      </h2>
      <ul>
        {todos.map(todo => (
          <TodoItem key={todo.id} todo={todo} />
        ))}
      </ul>
    </div>
  );
}
```

### Recoil Async Selectors

```typescript
// Async selector
const currentUserQuery = selector({
  key: 'currentUserQuery',
  get: async ({ get }) => {
    const userId = get(currentUserIdState);
    
    if (!userId) return null;
    
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  }
});

// Usage with Suspense
function CurrentUserInfo() {
  const currentUser = useRecoilValue(currentUserQuery);
  
  if (!currentUser) return <div>Please log in</div>;
  
  return <div>{currentUser.name}</div>;
}

function App() {
  return (
    <RecoilRoot>
      <React.Suspense fallback={<div>Loading...</div>}>
        <CurrentUserInfo />
      </React.Suspense>
    </RecoilRoot>
  );
}

// Selector with dependencies
const todoQuery = selector({
  key: 'todoQuery',
  get: async ({ get }) => {
    const filter = get(todoListFilterState);
    const search = get(searchTermState);
    
    const response = await fetch(
      `/api/todos?filter=${filter}&search=${search}`
    );
    return response.json();
  }
});
```

### Recoil Atom Families

```typescript
import { atomFamily, selectorFamily } from 'recoil';

// Atom family for individual todo items
const todoItemState = atomFamily<Todo, string>({
  key: 'todoItemState',
  default: (id) => ({
    id,
    text: '',
    completed: false
  })
});

// Usage
function TodoItem({ id }: { id: string }) {
  const [todo, setTodo] = useRecoilState(todoItemState(id));
  
  const handleToggle = () => {
    setTodo({ ...todo, completed: !todo.completed });
  };
  
  return (
    <li>
      <input type="checkbox" checked={todo.completed} onChange={handleToggle} />
      {todo.text}
    </li>
  );
}

// Selector family for async queries
const userInfoQuery = selectorFamily({
  key: 'userInfoQuery',
  get: (userId: string) => async () => {
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  }
});

function UserInfo({ userId }: { userId: string }) {
  const user = useRecoilValue(userInfoQuery(userId));
  return <div>{user.name}</div>;
}
```

### Recoil Persistence

```typescript
import { atom, useRecoilState } from 'recoil';
import { recoilPersist } from 'recoil-persist';

const { persistAtom } = recoilPersist({
  key: 'recoil-persist',
  storage: localStorage
});

// Atom with persistence
const userPreferencesState = atom({
  key: 'userPreferences',
  default: {
    theme: 'light',
    language: 'en'
  },
  effects_UNSTABLE: [persistAtom]
});

// Alternative: Manual persistence
const themeState = atom({
  key: 'themeState',
  default: 'light',
  effects_UNSTABLE: [
    ({ onSet, setSelf }) => {
      // Load from localStorage on init
      const savedValue = localStorage.getItem('theme');
      if (savedValue != null) {
        setSelf(savedValue);
      }
      
      // Save to localStorage on change
      onSet((newValue) => {
        localStorage.setItem('theme', newValue);
      });
    }
  ]
});
```

## Comparison: Jotai vs Recoil

```
┌────────────────────┬─────────────────┬──────────────────┐
│ Feature            │ Jotai           │ Recoil           │
├────────────────────┼─────────────────┼──────────────────┤
│ Bundle Size        │ ~3KB            │ ~20KB            │
│ API Surface        │ Minimal         │ Comprehensive    │
│ Atom Keys          │ Not required    │ Required         │
│ TypeScript         │ Excellent       │ Good             │
│ DevTools           │ Basic           │ Advanced         │
│ Persistence        │ Via utils       │ Via effects      │
│ Learning Curve     │ Easy            │ Moderate         │
│ React Suspense     │ First-class     │ First-class      │
│ Atom Families      │ Via utils       │ Built-in         │
│ Snapshots          │ Limited         │ Full support     │
└────────────────────┴─────────────────┴──────────────────┘

Choose Jotai when:
- Want minimal bundle size
- Prefer simpler API
- Don't need complex atom keys
- TypeScript is important

Choose Recoil when:
- Need advanced DevTools
- Want snapshot/time-travel
- Prefer explicit atom keys
- Building complex apps
```

## Advanced Patterns

### 1. Computed Atom Chains

```typescript
// Jotai example
const priceAtom = atom(100);
const quantityAtom = atom(2);
const taxRateAtom = atom(0.1);

// Chain of derived atoms
const subtotalAtom = atom((get) => 
  get(priceAtom) * get(quantityAtom)
);

const taxAtom = atom((get) => 
  get(subtotalAtom) * get(taxRateAtom)
);

const totalAtom = atom((get) => 
  get(subtotalAtom) + get(taxAtom)
);

// Component uses final computed value
function OrderTotal() {
  const total = useAtomValue(totalAtom);
  const tax = useAtomValue(taxAtom);
  
  return (
    <div>
      <p>Tax: ${tax.toFixed(2)}</p>
      <p>Total: ${total.toFixed(2)}</p>
    </div>
  );
}
```

### 2. Atom-based Routing

```typescript
// Jotai route atoms
const routeAtom = atom('/');
const paramsAtom = atom<Record<string, string>>({});

const currentRouteAtom = atom(
  (get) => ({
    path: get(routeAtom),
    params: get(paramsAtom)
  }),
  (get, set, update: { path: string; params?: Record<string, string> }) => {
    set(routeAtom, update.path);
    set(paramsAtom, update.params || {});
    window.history.pushState({}, '', update.path);
  }
);

function Router() {
  const [route, setRoute] = useAtom(currentRouteAtom);
  
  useEffect(() => {
    const handlePopState = () => {
      setRoute({ path: window.location.pathname });
    };
    
    window.addEventListener('popstate', handlePopState);
    return () => window.removeEventListener('popstate', handlePopState);
  }, []);
  
  switch (route.path) {
    case '/':
      return <Home />;
    case '/about':
      return <About />;
    default:
      return <NotFound />;
  }
}
```

### 3. Atom-based Forms

```typescript
import { atomFamily } from 'jotai/utils';

// Create atom for each form field
const fieldAtomFamily = atomFamily((fieldName: string) =>
  atom<{ value: string; error: string | null }>({
    value: '',
    error: null
  })
);

// Form validation atom
const formValidAtom = atom((get) => {
  const email = get(fieldAtomFamily('email'));
  const password = get(fieldAtomFamily('password'));
  
  return !email.error && !password.error && email.value && password.value;
});

function FormField({ name, label, validate }: {
  name: string;
  label: string;
  validate: (value: string) => string | null;
}) {
  const [field, setField] = useAtom(fieldAtomFamily(name));
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    const error = validate(value);
    setField({ value, error });
  };
  
  return (
    <div>
      <label>{label}</label>
      <input value={field.value} onChange={handleChange} />
      {field.error && <span className="error">{field.error}</span>}
    </div>
  );
}

function LoginForm() {
  const isValid = useAtomValue(formValidAtom);
  const email = useAtomValue(fieldAtomFamily('email'));
  const password = useAtomValue(fieldAtomFamily('password'));
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (isValid) {
      console.log('Submit:', email.value, password.value);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <FormField
        name="email"
        label="Email"
        validate={(v) => /\S+@\S+\.\S+/.test(v) ? null : 'Invalid email'}
      />
      <FormField
        name="password"
        label="Password"
        validate={(v) => v.length >= 8 ? null : 'Min 8 characters'}
      />
      <button type="submit" disabled={!isValid}>Login</button>
    </form>
  );
}
```

### 4. Optimistic Updates Pattern

```typescript
import { atom } from 'jotai';
import { atomWithStorage } from 'jotai/utils';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  _optimistic?: boolean;
}

// Persisted todos
const todosAtom = atomWithStorage<Todo[]>('todos', []);

// Atom for creating todo with optimistic update
const createTodoAtom = atom(
  null,
  async (get, set, text: string) => {
    const tempId = `temp_${Date.now()}`;
    const optimisticTodo: Todo = {
      id: tempId,
      text,
      completed: false,
      _optimistic: true
    };
    
    // Optimistically add to UI
    set(todosAtom, [...get(todosAtom), optimisticTodo]);
    
    try {
      // Send to server
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ text })
      });
      const serverTodo = await response.json();
      
      // Replace optimistic todo with server response
      set(todosAtom, (todos) =>
        todos.map(t => (t.id === tempId ? serverTodo : t))
      );
    } catch (error) {
      // Rollback on error
      set(todosAtom, (todos) => todos.filter(t => t.id !== tempId));
      throw error;
    }
  }
);

function TodoList() {
  const todos = useAtomValue(todosAtom);
  const createTodo = useSetAtom(createTodoAtom);
  
  const handleAdd = async (text: string) => {
    try {
      await createTodo(text);
    } catch (error) {
      alert('Failed to create todo');
    }
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id} className={todo._optimistic ? 'optimistic' : ''}>
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

## Common Mistakes

### 1. Creating Atoms Inside Components

```typescript
// Bad: Creates new atom on every render
function BadComponent() {
  const countAtom = atom(0); // ❌ New atom each render!
  const [count, setCount] = useAtom(countAtom);
  return <div>{count}</div>;
}

// Good: Define atoms outside components
const countAtom = atom(0);

function GoodComponent() {
  const [count, setCount] = useAtom(countAtom); // ✅ Same atom
  return <div>{count}</div>;
}
```

### 2. Not Using Read-Only Hooks

```typescript
// Bad: Component gets setCount even though it never uses it
function DisplayCount() {
  const [count, setCount] = useAtom(countAtom); // ❌ Unused setCount
  return <div>{count}</div>;
}

// Good: Use read-only hook
function DisplayCount() {
  const count = useAtomValue(countAtom); // ✅ Only gets value
  return <div>{count}</div>;
}
```

### 3. Over-Atomizing State

```typescript
// Bad: Too granular
const firstNameAtom = atom('');
const lastNameAtom = atom('');
const emailAtom = atom('');
const phoneAtom = atom('');
// 20 more atoms for one form...

// Good: Group related state
const userFormAtom = atom({
  firstName: '',
  lastName: '',
  email: '',
  phone: ''
});
```

### 4. Not Leveraging Derived Atoms

```typescript
// Bad: Manual recomputation
function TodoList() {
  const todos = useAtomValue(todosAtom);
  const filter = useAtomValue(filterAtom);
  
  // Recomputes on every render
  const filtered = todos.filter(t => {
    if (filter === 'active') return !t.completed;
    if (filter === 'completed') return t.completed;
    return true;
  });
  
  return <ul>{filtered.map(...)}</ul>;
}

// Good: Derived atom with automatic memoization
const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(filterAtom);
  // Only recomputes when todos or filter changes
  return todos.filter(...);
});

function TodoList() {
  const filtered = useAtomValue(filteredTodosAtom);
  return <ul>{filtered.map(...)}</ul>;
}
```

### 5. Forgetting RecoilRoot

```typescript
// Bad: No RecoilRoot
function App() {
  return <MyComponent />; // ❌ useRecoilState throws error
}

// Good: Wrap in RecoilRoot
function App() {
  return (
    <RecoilRoot>
      <MyComponent />
    </RecoilRoot>
  );
}
```

## Best Practices

### 1. Keep Atoms Small and Focused

```typescript
// Good: Small, focused atoms
const userIdAtom = atom<string | null>(null);
const userNameAtom = atom('');
const userEmailAtom = atom('');

// Components subscribe only to what they need
function UserName() {
  const name = useAtomValue(userNameAtom); // Only re-renders on name change
  return <div>{name}</div>;
}
```

### 2. Use Atom Families for Dynamic Data

```typescript
// Good: Atom family for dynamic collections
const todoAtomFamily = atomFamily((id: string) =>
  atom<Todo>({ id, text: '', completed: false })
);

// Each todo has its own atom
function TodoItem({ id }: { id: string }) {
  const [todo, setTodo] = useAtom(todoAtomFamily(id));
  // Only re-renders when this specific todo changes
}
```

### 3. Leverage Derived Atoms for Computed State

```typescript
// Good: Derived atoms for all computed values
const subtotalAtom = atom((get) => 
  get(cartItemsAtom).reduce((sum, item) => sum + item.price, 0)
);

const taxAtom = atom((get) => get(subtotalAtom) * 0.1);
const totalAtom = atom((get) => get(subtotalAtom) + get(taxAtom));
```

### 4. Use TypeScript for Type Safety

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// Good: Typed atoms
const userAtom = atom<User | null>(null);
const usersAtom = atom<User[]>([]);

// TypeScript ensures type safety
function UserProfile() {
  const user = useAtomValue(userAtom);
  if (user) {
    return <div>{user.name}</div>; // ✅ Type-safe
  }
  return null;
}
```

### 5. Split Read and Write for Performance

```typescript
// Good: Separate read and write when appropriate
function DisplayCount() {
  const count = useAtomValue(countAtom); // Read-only
  return <div>{count}</div>;
}

function IncrementButton() {
  const setCount = useSetAtom(countAtom); // Write-only
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

## When to Use Atom-Based State

### Good Use Cases

- **Fine-grained reactivity needed**: Only specific components should re-render
- **Distributed state**: State naturally exists in many independent pieces
- **Dynamic state**: Need to create state atoms on the fly
- **Form state**: Each field as independent atom
- **Performance-critical apps**: Minimize unnecessary re-renders
- **React Suspense integration**: Async atoms work seamlessly

### When NOT to Use

- **Simple applications**: Context or useState might be simpler
- **Strong Redux patterns**: Team familiar with Redux might prefer it
- **Need middleware**: Redux middleware ecosystem is mature
- **Time-travel debugging**: Redux DevTools more powerful
- **Server state**: React Query/SWR might be better suited

## Interview Questions

### Q1: What is an atom in atom-based state management?

**Answer:** An atom is the smallest, independent unit of state that can be subscribed to and updated separately. Unlike Redux's single store, atoms are distributed across the application. Each atom:
- Has its own value
- Can be read and written independently
- Notifies only the components that subscribe to it
- Can be composed into derived atoms

Example:
```typescript
const countAtom = atom(0);
const doubleAtom = atom((get) => get(countAtom) * 2);
```

Components subscribing to `doubleAtom` only re-render when `countAtom` changes, not when any other state changes.

### Q2: How does atom-based state improve performance?

**Answer:** Through granular subscriptions:

**Traditional Context/Redux:**
```typescript
// Component subscribes to entire state
const state = useSelector(state => state);
// Re-renders when ANY part of state changes
```

**Atom-based:**
```typescript
// Component subscribes only to specific atom
const user = useAtomValue(userAtom);
// Only re-renders when userAtom changes
```

Benefits:
1. Components re-render only when their specific atoms change
2. No need for complex memoization/selectors
3. Automatic optimization of derived state
4. Smaller subscription footprint

### Q3: What's the difference between Jotai and Recoil?

**Answer:**

**Jotai:**
- Minimal API (3KB)
- No required atom keys
- Simpler mental model
- Better TypeScript support
- Basic DevTools

**Recoil:**
- More features (20KB)
- Requires unique atom keys
- Advanced DevTools
- Snapshot/time-travel support
- More mature ecosystem

Choose Jotai for simplicity and bundle size, Recoil for advanced features and tooling.

### Q4: How do derived atoms work?

**Answer:** Derived atoms (selectors) automatically recompute when their dependencies change:

```typescript
// Base atoms
const todosAtom = atom([...]);
const filterAtom = atom('all');

// Derived atom
const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(filterAtom);
  return todos.filter(...);
});

// Automatically recomputes when todos or filter changes
// Memoized - only recomputes when dependencies change
```

Dependencies are tracked automatically by the `get` function. No manual dependency arrays needed.

### Q5: When should you use atom families?

**Answer:** Use atom families when you need to create atoms dynamically with parameters:

```typescript
// Atom family for individual items
const todoAtom = atomFamily((id: string) => 
  atom({ id, text: '', completed: false })
);

// Each ID gets its own atom
<TodoItem id="1" /> // Uses todoAtom("1")
<TodoItem id="2" /> // Uses todoAtom("2")
```

Good for:
- Collections where items update independently
- Dynamic form fields
- Per-item caching
- Route-based state

Don't use for static collections - use a single atom with an array instead.

### Q6: How do you handle async operations with atoms?

**Answer:** Create async atoms that return promises:

```typescript
// Async atom
const userAtom = atom(async (get) => {
  const userId = get(userIdAtom);
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
});

// Use with Suspense
function UserProfile() {
  const user = useAtomValue(userAtom); // Suspends while loading
  return <div>{user.name}</div>;
}

<Suspense fallback={<Loading />}>
  <UserProfile />
</Suspense>
```

Atoms integrate natively with React Suspense for async data fetching.

### Q7: How do you implement optimistic updates with atoms?

**Answer:**

```typescript
const updateTodoAtom = atom(
  null,
  async (get, set, { id, changes }) => {
    const todos = get(todosAtom);
    
    // Optimistically update
    set(todosAtom, todos.map(t => 
      t.id === id ? { ...t, ...changes } : t
    ));
    
    try {
      await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(changes)
      });
    } catch (error) {
      // Rollback on error
      set(todosAtom, todos);
      throw error;
    }
  }
);
```

Pattern:
1. Update atom immediately (optimistic)
2. Make API call
3. Keep update on success
4. Rollback on error

### Q8: What are the trade-offs of atom-based vs Redux?

**Answer:**

**Atom-based pros:**
- Fine-grained subscriptions (better performance)
- Less boilerplate
- Easier to learn
- More flexible state distribution
- Native async support

**Atom-based cons:**
- Less mature ecosystem
- Fewer middleware options
- Basic DevTools
- Less standardization

**Redux pros:**
- Mature ecosystem
- Powerful middleware
- Advanced DevTools
- Strong patterns/conventions
- Time-travel debugging

**Redux cons:**
- More boilerplate
- Coarser subscriptions
- Steeper learning curve
- Requires optimization for performance

Choose atom-based for new projects prioritizing performance and simplicity, Redux for large teams needing standardization and advanced tooling.

## Key Takeaways

- Atoms are independent, minimal units of state
- Fine-grained subscriptions prevent unnecessary re-renders
- Derived atoms automatically track dependencies and memoize
- Jotai is minimal (3KB), Recoil is feature-rich (20KB)
- Atom families create atoms dynamically with parameters
- Async atoms integrate natively with React Suspense
- Read-only hooks (useAtomValue) optimize performance
- Write-only hooks (useSetAtom) prevent unnecessary subscriptions
- Define atoms outside components to avoid recreating them
- Group related state, don't over-atomize
- Use TypeScript for type-safe atoms
- Atom-based state excels at fine-grained reactivity
- Better for distributed state vs. Redux's centralized store
- DevTools are less mature than Redux DevTools
- Great for performance-critical applications

## Resources

- [Jotai Documentation](https://jotai.org/)
- [Recoil Documentation](https://recoiljs.org/)
- [Jotai vs Recoil Comparison](https://blog.logrocket.com/jotai-vs-recoil-what-are-the-differences/)
- [Atomic State Management Guide](https://blog.logrocket.com/guide-to-atomic-state-management/)
- [Jotai Utilities](https://jotai.org/docs/utilities/introduction)
- [Recoil Best Practices](https://recoiljs.org/docs/guides/best-practices)
- [Fine-Grained Reactivity](https://dev.to/this-is-learning/fine-grained-reactivity-in-react-4bbc)
- [State Management Evolution](https://leerob.io/blog/react-state-management)
