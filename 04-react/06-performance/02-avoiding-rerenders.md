# Avoiding Unnecessary Re-renders

## The Idea

**In plain English:** When a webpage updates, React sometimes redraws parts of the screen that did not actually change — wasting time and making the app slower. Avoiding unnecessary re-renders means telling React exactly which parts need to redraw and which can stay as they are.

**Real-world analogy:** Imagine a restaurant kitchen where every time one chef updates the specials board, every single cook stops what they are doing, re-reads the entire menu, and starts over — even if their dish was completely unaffected. A smart kitchen would only notify the cooks whose recipes actually changed.

- The kitchen = the React component tree
- A cook being interrupted = a component re-rendering
- Only notifying affected cooks = memoization and context splitting

---

## Overview

Unnecessary re-renders are a common source of performance issues in React applications. Understanding when and why components re-render, and applying the right optimization techniques, is crucial for building performant applications. This guide covers context splitting, selector patterns, memoization strategies, and composition patterns to minimize re-renders.

## Table of Contents

1. [Understanding Re-renders](#understanding-re-renders)
2. [Context Splitting Patterns](#context-splitting-patterns)
3. [Selector Patterns](#selector-patterns)
4. [Memoization Strategies](#memoization-strategies)
5. [useMemo and useCallback](#usememo-and-usecallback)
6. [Composition Patterns](#composition-patterns)
7. [Component Structure](#component-structure)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Real-World Use Cases](#real-world-use-cases)
11. [Key Takeaways](#key-takeaways)
12. [Resources](#resources)

## Understanding Re-renders

Components re-render when state, props, or context changes:

### When Re-renders Happen

```typescript
import React, { useState } from 'react';

function App() {
  const [count, setCount] = useState(0);
  
  console.log('App rendered');
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Child re-renders even though it doesn't use count */}
      <Child />
    </div>
  );
}

function Child() {
  console.log('Child rendered');
  return <div>I am a child</div>;
}

// Output on button click:
// "App rendered"
// "Child rendered" ← Unnecessary!
```

### Re-render Causes

```typescript
import React, { useState, useContext, createContext } from 'react';

const ThemeContext = createContext('light');

function ExampleApp() {
  return <RenderCauses />;
}

function RenderCauses() {
  const [state, setState] = useState(0);
  const theme = useContext(ThemeContext);
  
  // Component re-renders when:
  // 1. State changes (setState called)
  // 2. Props change (parent passes different props)
  // 3. Context changes (ThemeContext value changes)
  // 4. Parent re-renders (by default, all children re-render)
  // 5. Force update (rarely used, not recommended)
  
  return <div>State: {state}, Theme: {theme}</div>;
}
```

## Context Splitting Patterns

Split contexts to minimize re-renders:

### Problem: Single Large Context

```typescript
import React, { createContext, useContext, useState } from 'react';

interface AppState {
  user: { name: string; email: string };
  theme: string;
  notifications: number;
  settings: { language: string; timezone: string };
}

const AppContext = createContext<{
  state: AppState;
  updateUser: (user: AppState['user']) => void;
  updateTheme: (theme: string) => void;
  updateNotifications: (count: number) => void;
  updateSettings: (settings: AppState['settings']) => void;
} | null>(null);

function AppProviderBad({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState<AppState>({
    user: { name: 'John', email: 'john@example.com' },
    theme: 'light',
    notifications: 0,
    settings: { language: 'en', timezone: 'UTC' },
  });
  
  // Problem: Any state change causes ALL consumers to re-render
  const value = {
    state,
    updateUser: (user: AppState['user']) => 
      setState(prev => ({ ...prev, user })),
    updateTheme: (theme: string) => 
      setState(prev => ({ ...prev, theme })),
    updateNotifications: (count: number) => 
      setState(prev => ({ ...prev, notifications: count })),
    updateSettings: (settings: AppState['settings']) => 
      setState(prev => ({ ...prev, settings })),
  };
  
  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}

// This component only needs theme, but re-renders when anything changes
function ThemedButton() {
  const context = useContext(AppContext);
  if (!context) throw new Error('Context required');
  
  return (
    <button style={{ background: context.state.theme === 'dark' ? '#333' : '#fff' }}>
      Click me
    </button>
  );
}
```

### Solution: Split Contexts

```typescript
import React, { createContext, useContext, useState } from 'react';

// Separate contexts for different concerns
const UserContext = createContext<{
  user: { name: string; email: string };
  updateUser: (user: { name: string; email: string }) => void;
} | null>(null);

const ThemeContext = createContext<{
  theme: string;
  updateTheme: (theme: string) => void;
} | null>(null);

const NotificationsContext = createContext<{
  count: number;
  updateCount: (count: number) => void;
} | null>(null);

function AppProviderGood({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState({ name: 'John', email: 'john@example.com' });
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState(0);
  
  return (
    <UserContext.Provider value={{ user, updateUser: setUser }}>
      <ThemeContext.Provider value={{ theme, updateTheme: setTheme }}>
        <NotificationsContext.Provider value={{ count: notifications, updateCount: setNotifications }}>
          {children}
        </NotificationsContext.Provider>
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}

// Now only re-renders when theme changes
function ThemedButton() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('Context required');
  
  console.log('ThemedButton rendered');
  
  return (
    <button style={{ background: context.theme === 'dark' ? '#333' : '#fff' }}>
      Click me
    </button>
  );
}

// Only re-renders when user changes
function UserProfile() {
  const context = useContext(UserContext);
  if (!context) throw new Error('Context required');
  
  console.log('UserProfile rendered');
  
  return <div>{context.user.name}</div>;
}
```

### Context with Selectors

```typescript
import React, { createContext, useContext, useState, useMemo, useRef, useSyncExternalStore } from 'react';

interface Store {
  user: { name: string; email: string };
  theme: string;
  notifications: number;
}

// Create a store with selector support
function createStore(initialState: Store) {
  let state = initialState;
  const listeners = new Set<() => void>();
  
  return {
    getState: () => state,
    setState: (newState: Partial<Store>) => {
      state = { ...state, ...newState };
      listeners.forEach(listener => listener());
    },
    subscribe: (listener: () => void) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

type StoreType = ReturnType<typeof createStore>;

const StoreContext = createContext<StoreType | null>(null);

function StoreProvider({ children }: { children: React.ReactNode }) {
  const store = useMemo(
    () => createStore({
      user: { name: 'John', email: 'john@example.com' },
      theme: 'light',
      notifications: 0,
    }),
    []
  );
  
  return <StoreContext.Provider value={store}>{children}</StoreContext.Provider>;
}

// Hook that accepts a selector
function useStore<T>(selector: (state: Store) => T): T {
  const store = useContext(StoreContext);
  if (!store) throw new Error('StoreContext required');
  
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}

// Only re-renders when theme changes
function ThemedComponent() {
  const theme = useStore(state => state.theme);
  
  console.log('ThemedComponent rendered');
  
  return <div>Theme: {theme}</div>;
}

// Only re-renders when user name changes
function UserName() {
  const userName = useStore(state => state.user.name);
  
  console.log('UserName rendered');
  
  return <div>Name: {userName}</div>;
}
```

## Selector Patterns

Use selectors to subscribe to specific data:

### Basic Selectors

```typescript
import React from 'react';
import { useStore } from './store'; // Zustand or similar

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

interface AppState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  addTodo: (text: string) => void;
  toggleTodo: (id: number) => void;
  setFilter: (filter: 'all' | 'active' | 'completed') => void;
}

// Only re-renders when filtered todos change
function TodoList() {
  const filteredTodos = useStore((state: AppState) => {
    if (state.filter === 'all') return state.todos;
    if (state.filter === 'active') return state.todos.filter(t => !t.completed);
    return state.todos.filter(t => t.completed);
  });
  
  console.log('TodoList rendered');
  
  return (
    <ul>
      {filteredTodos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

// Only re-renders when this specific todo changes
const TodoItem = React.memo(function TodoItem({ todo }: { todo: Todo }) {
  const toggleTodo = useStore((state: AppState) => state.toggleTodo);
  
  console.log(`TodoItem ${todo.id} rendered`);
  
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => toggleTodo(todo.id)}
      />
      {todo.text}
    </li>
  );
});
```

### Memoized Selectors

```typescript
import React, { useMemo } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

interface State {
  products: Product[];
  searchTerm: string;
  categoryFilter: string;
}

function ProductList() {
  // Problem: Creates new array every render, even if data unchanged
  const products = useStore((state: State) => 
    state.products
      .filter(p => p.name.includes(state.searchTerm))
      .filter(p => !state.categoryFilter || p.category === state.categoryFilter)
  );
  
  return <div>{products.length} products</div>;
}

// Solution: Memoize selector
function ProductListOptimized() {
  const products = useStore((state: State) => state.products);
  const searchTerm = useStore((state: State) => state.searchTerm);
  const categoryFilter = useStore((state: State) => state.categoryFilter);
  
  const filteredProducts = useMemo(
    () => products
      .filter(p => p.name.includes(searchTerm))
      .filter(p => !categoryFilter || p.category === categoryFilter),
    [products, searchTerm, categoryFilter]
  );
  
  return <div>{filteredProducts.length} products</div>;
}
```

## Memoization Strategies

Choose the right memoization approach:

### React.memo for Components

```typescript
import React, { useState } from 'react';

// Without memo: Re-renders on every parent render
function ExpensiveComponent({ value }: { value: number }) {
  console.log('ExpensiveComponent rendered');
  
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += Math.random();
  }
  
  return <div>Value: {value}, Result: {result}</div>;
}

// With memo: Only re-renders when value changes
const ExpensiveComponentMemo = React.memo(ExpensiveComponent);

function App() {
  const [count, setCount] = useState(0);
  const [value, setValue] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <button onClick={() => setValue(value + 1)}>
        Value: {value}
      </button>
      
      {/* Re-renders on every button click */}
      <ExpensiveComponent value={value} />
      
      {/* Only re-renders when value changes */}
      <ExpensiveComponentMemo value={value} />
    </div>
  );
}
```

### useMemo for Expensive Computations

```typescript
import React, { useState, useMemo } from 'react';

interface Item {
  id: number;
  name: string;
  value: number;
}

function DataAnalysis({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  
  // Without useMemo: Recalculates on every render
  const stats = {
    total: items.reduce((sum, item) => sum + item.value, 0),
    average: items.reduce((sum, item) => sum + item.value, 0) / items.length,
    max: Math.max(...items.map(item => item.value)),
    min: Math.min(...items.map(item => item.value)),
  };
  
  // With useMemo: Only recalculates when items change
  const statsMemo = useMemo(() => ({
    total: items.reduce((sum, item) => sum + item.value, 0),
    average: items.reduce((sum, item) => sum + item.value, 0) / items.length,
    max: Math.max(...items.map(item => item.value)),
    min: Math.min(...items.map(item => item.value)),
  }), [items]);
  
  return (
    <div>
      <input
        value={filter}
        onChange={e => setFilter(e.target.value)}
        placeholder="Filter (causes re-render)"
      />
      <div>Total: {statsMemo.total}</div>
      <div>Average: {statsMemo.average}</div>
      <div>Max: {statsMemo.max}</div>
      <div>Min: {statsMemo.min}</div>
    </div>
  );
}
```

### useCallback for Functions

```typescript
import React, { useState, useCallback } from 'react';

interface Item {
  id: number;
  name: string;
}

const ListItem = React.memo(function ListItem({
  item,
  onDelete,
}: {
  item: Item;
  onDelete: (id: number) => void;
}) {
  console.log(`ListItem ${item.id} rendered`);
  
  return (
    <div>
      {item.name}
      <button onClick={() => onDelete(item.id)}>Delete</button>
    </div>
  );
});

function List({ items }: { items: Item[] }) {
  const [count, setCount] = useState(0);
  
  // Without useCallback: New function every render
  // All ListItems re-render even though items unchanged
  const handleDelete = (id: number) => {
    console.log('Delete', id);
  };
  
  // With useCallback: Stable function reference
  // ListItems don't re-render when count changes
  const handleDeleteMemo = useCallback((id: number) => {
    console.log('Delete', id);
  }, []);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {items.map(item => (
        <ListItem 
          key={item.id} 
          item={item} 
          onDelete={handleDeleteMemo}
        />
      ))}
    </div>
  );
}
```

## useMemo and useCallback

Proper usage of memoization hooks:

### When to Use useMemo

```typescript
import React, { useMemo, useState } from 'react';

function ComponentWithUseMemo() {
  const [items, setItems] = useState<number[]>([]);
  const [filter, setFilter] = useState('');
  
  // GOOD: Expensive computation
  const sortedItems = useMemo(
    () => [...items].sort((a, b) => b - a),
    [items]
  );
  
  // GOOD: Complex object creation
  const config = useMemo(
    () => ({
      theme: 'dark',
      settings: { autoSave: true },
    }),
    []
  );
  
  // BAD: Simple computation (overhead not worth it)
  const doubleCount = useMemo(
    () => items.length * 2,
    [items.length]
  );
  
  // BETTER: Just compute directly
  const doubleCountBetter = items.length * 2;
  
  return <div>{sortedItems.length}</div>;
}
```

### When to Use useCallback

```typescript
import React, { useCallback, useState } from 'react';

function ComponentWithUseCallback() {
  const [items, setItems] = useState<string[]>([]);
  
  // GOOD: Passed to memoized child
  const handleAdd = useCallback((item: string) => {
    setItems(prev => [...prev, item]);
  }, []);
  
  // GOOD: Used in effect dependency array
  const handleFetch = useCallback(async () => {
    const response = await fetch('/api/items');
    const data = await response.json();
    setItems(data);
  }, []);
  
  React.useEffect(() => {
    handleFetch();
  }, [handleFetch]);
  
  // BAD: Not passed to child, no dependencies
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  // BETTER: Just use inline
  const handleClickBetter = () => {
    console.log('clicked');
  };
  
  return (
    <div>
      <MemoizedChild onAdd={handleAdd} />
      <button onClick={handleClickBetter}>Click</button>
    </div>
  );
}

const MemoizedChild = React.memo(function Child({ 
  onAdd 
}: { 
  onAdd: (item: string) => void 
}) {
  return <button onClick={() => onAdd('new')}>Add</button>;
});
```

### Combining Memoization Techniques

```typescript
import React, { useState, useMemo, useCallback } from 'react';

interface User {
  id: number;
  name: string;
  role: string;
}

function UserList({ users }: { users: User[] }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [selectedRole, setSelectedRole] = useState<string | null>(null);
  
  // Memoize filtered users
  const filteredUsers = useMemo(() => {
    return users.filter(user => {
      const matchesSearch = user.name
        .toLowerCase()
        .includes(searchTerm.toLowerCase());
      const matchesRole = !selectedRole || user.role === selectedRole;
      return matchesSearch && matchesRole;
    });
  }, [users, searchTerm, selectedRole]);
  
  // Memoize callback
  const handleUserClick = useCallback((userId: number) => {
    console.log('User clicked:', userId);
  }, []);
  
  return (
    <div>
      <input
        value={searchTerm}
        onChange={e => setSearchTerm(e.target.value)}
        placeholder="Search users"
      />
      
      <select
        value={selectedRole || ''}
        onChange={e => setSelectedRole(e.target.value || null)}
      >
        <option value="">All roles</option>
        <option value="admin">Admin</option>
        <option value="user">User</option>
      </select>
      
      {filteredUsers.map(user => (
        <UserCard
          key={user.id}
          user={user}
          onClick={handleUserClick}
        />
      ))}
    </div>
  );
}

const UserCard = React.memo(function UserCard({
  user,
  onClick,
}: {
  user: User;
  onClick: (userId: number) => void;
}) {
  console.log(`UserCard ${user.id} rendered`);
  
  return (
    <div onClick={() => onClick(user.id)}>
      {user.name} - {user.role}
    </div>
  );
});
```

## Composition Patterns

Use composition to avoid prop drilling and unnecessary renders:

### Children as Prop

```typescript
import React, { useState } from 'react';

// BAD: StaticContent re-renders when count changes
function AppBad() {
  const [count, setCount] = useState(0);
  
  return (
    <Container>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <StaticContent /> {/* Unnecessary re-render */}
    </Container>
  );
}

function Container({ children }: { children: React.ReactNode }) {
  return <div className="container">{children}</div>;
}

function StaticContent() {
  console.log('StaticContent rendered');
  return <div>Static content</div>;
}

// GOOD: StaticContent doesn't re-render
function AppGood() {
  return (
    <Container>
      <Counter />
      <StaticContent /> {/* No re-render */}
    </Container>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

### Lifting Content Up

```typescript
import React, { useState } from 'react';

// BAD: Entire modal content re-renders
function ModalBad() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && (
        <div className="modal">
          <ExpensiveContent /> {/* Re-renders on every toggle */}
        </div>
      )}
    </div>
  );
}

// GOOD: Content created once, lifted up
function ModalGood() {
  const [isOpen, setIsOpen] = useState(false);
  
  const content = <ExpensiveContent />; // Created once
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && (
        <div className="modal">
          {content} {/* Same instance every time */}
        </div>
      )}
    </div>
  );
}

function ExpensiveContent() {
  console.log('ExpensiveContent rendered');
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += Math.random();
  }
  return <div>Result: {result}</div>;
}
```

## Component Structure

Structure components to minimize re-renders:

### Colocate State

```typescript
import React, { useState } from 'react';

// BAD: State at top level causes all children to re-render
function AppBad() {
  const [formData, setFormData] = useState({ name: '', email: '' });
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <Counter count={count} setCount={setCount} />
      <Form formData={formData} setFormData={setFormData} />
      <StaticSidebar /> {/* Unnecessary re-render */}
    </div>
  );
}

// GOOD: State colocated with components that use it
function AppGood() {
  return (
    <div>
      <Counter />
      <Form />
      <StaticSidebar /> {/* No re-render */}
    </div>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}

function Form() {
  const [formData, setFormData] = useState({ name: '', email: '' });
  return <div>{/* form fields */}</div>;
}

function StaticSidebar() {
  console.log('StaticSidebar rendered');
  return <div>Sidebar</div>;
}
```

### Extract Components

```typescript
import React, { useState } from 'react';

// BAD: Entire component re-renders for form changes
function ProfileBad() {
  const [name, setName] = useState('');
  const [bio, setBio] = useState('');
  
  return (
    <div>
      <h1>Profile</h1>
      
      <input
        value={name}
        onChange={e => setName(e.target.value)}
      />
      
      <textarea
        value={bio}
        onChange={e => setBio(e.target.value)}
      />
      
      <ExpensiveStats /> {/* Re-renders on every keystroke */}
    </div>
  );
}

// GOOD: Extract form to separate component
function ProfileGood() {
  return (
    <div>
      <h1>Profile</h1>
      <ProfileForm />
      <ExpensiveStats /> {/* Doesn't re-render */}
    </div>
  );
}

function ProfileForm() {
  const [name, setName] = useState('');
  const [bio, setBio] = useState('');
  
  return (
    <>
      <input
        value={name}
        onChange={e => setName(e.target.value)}
      />
      <textarea
        value={bio}
        onChange={e => setBio(e.target.value)}
      />
    </>
  );
}

function ExpensiveStats() {
  console.log('ExpensiveStats rendered');
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += Math.random();
  }
  return <div>Stats: {result}</div>;
}
```

## Common Mistakes

### Over-Memoization

```typescript
import React, { useMemo, useCallback } from 'react';

// BAD: Over-memoized
function OverMemoized({ value }: { value: number }) {
  const doubled = useMemo(() => value * 2, [value]); // Unnecessary
  const tripled = useMemo(() => value * 3, [value]); // Unnecessary
  const handleClick = useCallback(() => {
    console.log(value);
  }, [value]); // Unnecessary
  
  return (
    <div>
      <div>{doubled}</div>
      <div>{tripled}</div>
      <button onClick={handleClick}>Click</button>
    </div>
  );
}

// GOOD: Only memoize when necessary
function Optimized({ value }: { value: number }) {
  const doubled = value * 2; // Simple computation
  const tripled = value * 3; // Simple computation
  
  return (
    <div>
      <div>{doubled}</div>
      <div>{tripled}</div>
      <button onClick={() => console.log(value)}>Click</button>
    </div>
  );
}
```

### Memoization Without Dependencies

```typescript
import React, { useMemo } from 'react';

function MemoMistakes({ items }: { items: number[] }) {
  // BAD: Missing dependency
  const sorted = useMemo(
    () => [...items].sort(),
    [] // Should include [items]
  );
  
  // BAD: Wrong dependencies
  const filtered = useMemo(
    () => items.filter(x => x > 5),
    [items.length] // Should be [items]
  );
  
  // GOOD: Correct dependencies
  const processed = useMemo(
    () => items.filter(x => x > 5).sort(),
    [items]
  );
  
  return <div>{processed.length}</div>;
}
```

## Best Practices

1. **Profile before optimizing**: Use React DevTools to identify issues
2. **Colocate state**: Keep state close to where it's used
3. **Extract components**: Separate static from dynamic parts
4. **Split contexts**: Avoid single large context
5. **Use selectors**: Subscribe to specific data only
6. **Memoize strategically**: Only for expensive operations
7. **Stable references**: Use useMemo/useCallback for object/function props
8. **Composition over props**: Use children prop to avoid re-renders
9. **React.memo carefully**: Only for pure components with frequent re-renders
10. **Measure impact**: Verify optimizations actually improve performance

## Real-World Use Cases

### Shopping Cart

```typescript
import React, { useState, useMemo, useCallback, createContext, useContext } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
}

interface CartItem extends Product {
  quantity: number;
}

// Split contexts
const CartItemsContext = createContext<CartItem[]>([]);
const CartActionsContext = createContext<{
  addItem: (product: Product) => void;
  removeItem: (id: number) => void;
  updateQuantity: (id: number, quantity: number) => void;
} | null>(null);

function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<CartItem[]>([]);
  
  const actions = useMemo(
    () => ({
      addItem: (product: Product) => {
        setItems(prev => {
          const existing = prev.find(item => item.id === product.id);
          if (existing) {
            return prev.map(item =>
              item.id === product.id
                ? { ...item, quantity: item.quantity + 1 }
                : item
            );
          }
          return [...prev, { ...product, quantity: 1 }];
        });
      },
      removeItem: (id: number) => {
        setItems(prev => prev.filter(item => item.id !== id));
      },
      updateQuantity: (id: number, quantity: number) => {
        setItems(prev =>
          prev.map(item =>
            item.id === id ? { ...item, quantity } : item
          )
        );
      },
    }),
    []
  );
  
  return (
    <CartItemsContext.Provider value={items}>
      <CartActionsContext.Provider value={actions}>
        {children}
      </CartActionsContext.Provider>
    </CartItemsContext.Provider>
  );
}

// Only re-renders when cart items change
function CartSummary() {
  const items = useContext(CartItemsContext);
  
  const total = useMemo(
    () => items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  );
  
  const itemCount = useMemo(
    () => items.reduce((sum, item) => sum + item.quantity, 0),
    [items]
  );
  
  console.log('CartSummary rendered');
  
  return (
    <div>
      <div>Items: {itemCount}</div>
      <div>Total: ${total}</div>
    </div>
  );
}

// Only individual item re-renders when its quantity changes
const CartItem = React.memo(function CartItem({ item }: { item: CartItem }) {
  const actions = useContext(CartActionsContext);
  if (!actions) throw new Error('CartActionsContext required');
  
  console.log(`CartItem ${item.id} rendered`);
  
  return (
    <div>
      <span>{item.name}</span>
      <span>${item.price}</span>
      <input
        type="number"
        value={item.quantity}
        onChange={e => actions.updateQuantity(item.id, parseInt(e.target.value))}
      />
      <button onClick={() => actions.removeItem(item.id)}>Remove</button>
    </div>
  );
});

function Cart() {
  const items = useContext(CartItemsContext);
  
  return (
    <div>
      {items.map(item => (
        <CartItem key={item.id} item={item} />
      ))}
      <CartSummary />
    </div>
  );
}
```

## Key Takeaways

1. **Understand re-render causes**: State, props, context, parent re-renders
2. **Split contexts**: Separate concerns to minimize consumer re-renders
3. **Use selectors**: Subscribe to specific data slices
4. **React.memo for components**: Prevent re-renders when props unchanged
5. **useMemo for values**: Cache expensive computations
6. **useCallback for functions**: Stable function references
7. **Composition patterns**: Use children prop to skip re-renders
8. **Colocate state**: Keep state close to components that use it
9. **Extract components**: Separate dynamic from static parts
10. **Profile and measure**: Optimize based on actual performance data

## Resources

- [Optimizing Performance](https://react.dev/learn/render-and-commit)
- [React.memo](https://react.dev/reference/react/memo)
- [useMemo](https://react.dev/reference/react/useMemo)
- [useCallback](https://react.dev/reference/react/useCallback)
- [Before You memo()](https://overreacted.io/before-you-memo/)
- [React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/)
