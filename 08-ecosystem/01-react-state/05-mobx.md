# MobX - Simple, Scalable State Management

## Overview

MobX is a battle-tested library that makes state management simple and scalable by transparently applying functional reactive programming (TFRP). The philosophy is straightforward: anything that can be derived from application state should be derived automatically.

### Key Features

- **Simple and straightforward**: Minimal boilerplate
- **Reactive**: Automatic dependency tracking
- **Observable**: Make any data structure observable
- **Computed values**: Derived state automatically updated
- **Actions**: Modify state in transactions
- **Flexible**: Use with classes or plain objects
- **Framework agnostic**: Works with React, Vue, Angular

## Installation and Setup

```bash
# npm
npm install mobx mobx-react-lite

# yarn
yarn add mobx mobx-react-lite

# pnpm
pnpm add mobx mobx-react-lite
```

## Observables

### makeObservable

```typescript
import { makeObservable, observable, action, computed } from 'mobx';

class CounterStore {
  count = 0;

  constructor() {
    makeObservable(this, {
      count: observable,
      increment: action,
      decrement: action,
      reset: action,
      doubled: computed,
    });
  }

  increment() {
    this.count++;
  }

  decrement() {
    this.count--;
  }

  reset() {
    this.count = 0;
  }

  get doubled() {
    return this.count * 2;
  }
}

// Usage with React
import { observer } from 'mobx-react-lite';

const counterStore = new CounterStore();

const Counter = observer(() => {
  return (
    <div>
      <p>Count: {counterStore.count}</p>
      <p>Doubled: {counterStore.doubled}</p>
      <button onClick={() => counterStore.increment()}>+</button>
      <button onClick={() => counterStore.decrement()}>-</button>
      <button onClick={() => counterStore.reset()}>Reset</button>
    </div>
  );
});
```

### makeAutoObservable

```typescript
import { makeAutoObservable } from 'mobx';

class TodoStore {
  todos: Array<{ id: string; text: string; completed: boolean }> = [];
  filter: 'all' | 'active' | 'completed' = 'all';

  constructor() {
    makeAutoObservable(this);
  }

  addTodo(text: string) {
    this.todos.push({
      id: crypto.randomUUID(),
      text,
      completed: false,
    });
  }

  toggleTodo(id: string) {
    const todo = this.todos.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
    }
  }

  deleteTodo(id: string) {
    this.todos = this.todos.filter(t => t.id !== id);
  }

  setFilter(filter: 'all' | 'active' | 'completed') {
    this.filter = filter;
  }

  get filteredTodos() {
    switch (this.filter) {
      case 'active':
        return this.todos.filter(t => !t.completed);
      case 'completed':
        return this.todos.filter(t => t.completed);
      default:
        return this.todos;
    }
  }

  get stats() {
    return {
      total: this.todos.length,
      completed: this.todos.filter(t => t.completed).length,
      active: this.todos.filter(t => !t.completed).length,
    };
  }
}

const todoStore = new TodoStore();

const TodoApp = observer(() => {
  const [newTodo, setNewTodo] = React.useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (newTodo.trim()) {
      todoStore.addTodo(newTodo);
      setNewTodo('');
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input value={newTodo} onChange={e => setNewTodo(e.target.value)} />
        <button type="submit">Add</button>
      </form>

      <div>
        <button onClick={() => todoStore.setFilter('all')}>All</button>
        <button onClick={() => todoStore.setFilter('active')}>Active</button>
        <button onClick={() => todoStore.setFilter('completed')}>Completed</button>
      </div>

      <ul>
        {todoStore.filteredTodos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => todoStore.toggleTodo(todo.id)}
            />
            {todo.text}
            <button onClick={() => todoStore.deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>

      <div>
        <p>Total: {todoStore.stats.total}</p>
        <p>Active: {todoStore.stats.active}</p>
        <p>Completed: {todoStore.stats.completed}</p>
      </div>
    </div>
  );
});
```

## Computed Values

### Derived State

```typescript
import { makeAutoObservable } from 'mobx';

class ShoppingCart {
  items: Array<{ id: string; name: string; price: number; quantity: number }> = [];
  taxRate = 0.1;

  constructor() {
    makeAutoObservable(this);
  }

  addItem(name: string, price: number) {
    const existing = this.items.find(i => i.name === name);
    if (existing) {
      existing.quantity++;
    } else {
      this.items.push({ id: crypto.randomUUID(), name, price, quantity: 1 });
    }
  }

  removeItem(id: string) {
    this.items = this.items.filter(i => i.id !== id);
  }

  updateQuantity(id: string, quantity: number) {
    const item = this.items.find(i => i.id === id);
    if (item) {
      item.quantity = quantity;
    }
  }

  get subtotal() {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  get tax() {
    return this.subtotal * this.taxRate;
  }

  get total() {
    return this.subtotal + this.tax;
  }

  get itemCount() {
    return this.items.reduce((sum, item) => sum + item.quantity, 0);
  }
}

const cart = new ShoppingCart();

const Cart = observer(() => {
  return (
    <div>
      <h2>Shopping Cart ({cart.itemCount} items)</h2>
      <ul>
        {cart.items.map(item => (
          <li key={item.id}>
            {item.name} - ${item.price} x {item.quantity}
            <button onClick={() => cart.updateQuantity(item.id, item.quantity + 1)}>+</button>
            <button onClick={() => cart.updateQuantity(item.id, Math.max(0, item.quantity - 1))}>-</button>
            <button onClick={() => cart.removeItem(item.id)}>Remove</button>
          </li>
        ))}
      </ul>
      <div>
        <p>Subtotal: ${cart.subtotal.toFixed(2)}</p>
        <p>Tax: ${cart.tax.toFixed(2)}</p>
        <p><strong>Total: ${cart.total.toFixed(2)}</strong></p>
      </div>
    </div>
  );
});
```

## Actions

### Bound Actions

```typescript
import { makeAutoObservable, runInAction } from 'mobx';

class UserStore {
  user: { id: string; name: string; email: string } | null = null;
  loading = false;
  error: string | null = null;

  constructor() {
    makeAutoObservable(this);
  }

  async fetchUser(userId: string) {
    this.loading = true;
    this.error = null;

    try {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error('Failed to fetch user');
      const data = await response.json();

      runInAction(() => {
        this.user = data;
        this.loading = false;
      });
    } catch (error) {
      runInAction(() => {
        this.error = (error as Error).message;
        this.loading = false;
      });
    }
  }

  async updateUser(updates: Partial<{ name: string; email: string }>) {
    if (!this.user) return;

    this.loading = true;
    this.error = null;

    try {
      const response = await fetch(`/api/users/${this.user.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates),
      });

      if (!response.ok) throw new Error('Failed to update user');
      const data = await response.json();

      runInAction(() => {
        this.user = data;
        this.loading = false;
      });
    } catch (error) {
      runInAction(() => {
        this.error = (error as Error).message;
        this.loading = false;
      });
    }
  }

  logout() {
    this.user = null;
    this.error = null;
  }
}

const userStore = new UserStore();

const UserProfile = observer(() => {
  React.useEffect(() => {
    userStore.fetchUser('123');
  }, []);

  if (userStore.loading) return <div>Loading...</div>;
  if (userStore.error) return <div>Error: {userStore.error}</div>;
  if (!userStore.user) return <div>No user</div>;

  return (
    <div>
      <h2>{userStore.user.name}</h2>
      <p>{userStore.user.email}</p>
      <button onClick={() => userStore.updateUser({ name: 'New Name' })}>
        Update Name
      </button>
      <button onClick={() => userStore.logout()}>Logout</button>
    </div>
  );
});
```

## Reactions

### autorun

```typescript
import { makeAutoObservable, autorun } from 'mobx';

class ThemeStore {
  theme: 'light' | 'dark' = 'light';

  constructor() {
    makeAutoObservable(this);

    // Automatically persist theme changes
    autorun(() => {
      localStorage.setItem('theme', this.theme);
      document.body.className = this.theme;
    });
  }

  toggleTheme() {
    this.theme = this.theme === 'light' ? 'dark' : 'light';
  }
}

const themeStore = new ThemeStore();
```

### reaction

```typescript
import { makeAutoObservable, reaction } from 'mobx';

class NotificationStore {
  notifications: Array<{ id: string; message: string; type: 'info' | 'success' | 'error' }> = [];

  constructor() {
    makeAutoObservable(this);

    // React to notification changes
    reaction(
      () => this.notifications.length,
      (length) => {
        if (length > 0) {
          console.log(`You have ${length} notifications`);
        }
      }
    );
  }

  addNotification(message: string, type: 'info' | 'success' | 'error') {
    const id = crypto.randomUUID();
    this.notifications.push({ id, message, type });

    // Auto-remove after 3 seconds
    setTimeout(() => {
      this.removeNotification(id);
    }, 3000);
  }

  removeNotification(id: string) {
    this.notifications = this.notifications.filter(n => n.id !== id);
  }
}
```

### when

```typescript
import { makeAutoObservable, when } from 'mobx';

class AuthStore {
  isAuthenticated = false;
  user: any = null;

  constructor() {
    makeAutoObservable(this);

    // Wait for authentication
    when(
      () => this.isAuthenticated,
      () => {
        console.log('User authenticated, loading profile...');
        this.loadUserProfile();
      }
    );
  }

  login(user: any) {
    this.user = user;
    this.isAuthenticated = true;
  }

  loadUserProfile() {
    // Load user profile
  }
}
```

## Observable Collections

### Arrays

```typescript
import { makeAutoObservable } from 'mobx';

class ListStore {
  items = ['Apple', 'Banana', 'Cherry'];

  constructor() {
    makeAutoObservable(this);
  }

  addItem(item: string) {
    this.items.push(item);
  }

  removeItem(index: number) {
    this.items.splice(index, 1);
  }

  clear() {
    this.items.clear(); // MobX observable arrays have .clear()
  }

  get sortedItems() {
    return this.items.slice().sort();
  }
}
```

### Maps

```typescript
import { makeAutoObservable, observable } from 'mobx';

class UserCache {
  users = observable.map<string, { id: string; name: string }>();

  constructor() {
    makeAutoObservable(this, {
      users: observable,
    });
  }

  addUser(id: string, user: { id: string; name: string }) {
    this.users.set(id, user);
  }

  getUser(id: string) {
    return this.users.get(id);
  }

  removeUser(id: string) {
    this.users.delete(id);
  }

  get userCount() {
    return this.users.size;
  }

  get allUsers() {
    return Array.from(this.users.values());
  }
}
```

### Sets

```typescript
import { makeAutoObservable, observable } from 'mobx';

class TagStore {
  tags = observable.set<string>();

  constructor() {
    makeAutoObservable(this, {
      tags: observable,
    });
  }

  addTag(tag: string) {
    this.tags.add(tag);
  }

  removeTag(tag: string) {
    this.tags.delete(tag);
  }

  hasTag(tag: string) {
    return this.tags.has(tag);
  }

  get tagCount() {
    return this.tags.size;
  }

  get tagList() {
    return Array.from(this.tags).sort();
  }
}
```

## Root Store Pattern

```typescript
import { makeAutoObservable } from 'mobx';
import { createContext, useContext } from 'react';

class UserStore {
  user: any = null;

  constructor(private rootStore: RootStore) {
    makeAutoObservable(this);
  }

  setUser(user: any) {
    this.user = user;
  }
}

class TodoStore {
  todos: any[] = [];

  constructor(private rootStore: RootStore) {
    makeAutoObservable(this);
  }

  addTodo(text: string) {
    this.todos.push({ id: crypto.randomUUID(), text, completed: false });
  }
}

class RootStore {
  userStore: UserStore;
  todoStore: TodoStore;

  constructor() {
    this.userStore = new UserStore(this);
    this.todoStore = new TodoStore(this);
  }
}

const rootStore = new RootStore();
const StoreContext = createContext(rootStore);

export const useStore = () => useContext(StoreContext);

// Usage
const TodoList = observer(() => {
  const { todoStore, userStore } = useStore();
  
  return (
    <div>
      <p>User: {userStore.user?.name}</p>
      <ul>
        {todoStore.todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </div>
  );
});
```

## Flow (Async Actions)

```typescript
import { makeAutoObservable, flow } from 'mobx';

class DataStore {
  data: any[] = [];
  loading = false;
  error: string | null = null;

  constructor() {
    makeAutoObservable(this, {
      fetchData: flow,
    });
  }

  *fetchData() {
    this.loading = true;
    this.error = null;

    try {
      const response = yield fetch('/api/data');
      if (!response.ok) throw new Error('Failed to fetch');
      this.data = yield response.json();
    } catch (error) {
      this.error = (error as Error).message;
    } finally {
      this.loading = false;
    }
  }
}

const store = new DataStore();

const DataList = observer(() => {
  React.useEffect(() => {
    store.fetchData();
  }, []);

  if (store.loading) return <div>Loading...</div>;
  if (store.error) return <div>Error: {store.error}</div>;

  return (
    <ul>
      {store.data.map((item: any) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
});
```

## Testing

```typescript
import { makeAutoObservable } from 'mobx';

class Counter {
  count = 0;

  constructor() {
    makeAutoObservable(this);
  }

  increment() {
    this.count++;
  }

  decrement() {
    this.count--;
  }
}

describe('Counter', () => {
  it('should increment', () => {
    const counter = new Counter();
    counter.increment();
    expect(counter.count).toBe(1);
  });

  it('should decrement', () => {
    const counter = new Counter();
    counter.decrement();
    expect(counter.count).toBe(-1);
  });
});
```

## Common Mistakes

1. **Not using observer**: Components must be wrapped with observer
2. **Modifying state outside actions**: Use actions for all state changes
3. **Not using runInAction**: For async updates after await
4. **Over-observing**: Too many computed values can hurt performance
5. **Mutating observables directly in render**: Always use actions

## Best Practices

1. **Use makeAutoObservable**: Simpler than makeObservable
2. **Keep stores focused**: Single responsibility principle
3. **Use computed for derived state**: Automatic memoization
4. **Actions for mutations**: Keep state changes predictable
5. **Use reactions sparingly**: For side effects only
6. **Root store pattern**: Organize multiple stores
7. **TypeScript**: Full type safety

## When to Use MobX

### Good Use Cases
- Applications with complex state
- Need automatic reactivity
- OOP-style code preferred
- Rapid development
- Less boilerplate desired

### Not Ideal For
- Simple applications
- When immutability is required
- Team unfamiliar with reactive programming

## Interview Questions

1. **What is MobX's core concept?**
   - Automatic reactive updates based on observable state

2. **What is the difference between observable and computed?**
   - Observable is mutable state; computed is derived read-only state

3. **When should you use actions?**
   - For any code that modifies observable state

4. **What is runInAction used for?**
   - Wrapping async state updates after await

5. **How does observer work?**
   - Makes React components reactive to observable changes

## Key Takeaways

- MobX provides automatic reactive state management
- Observable state tracked automatically
- Computed values derived automatically
- Actions modify state predictably
- Reactions for side effects
- Less boilerplate than Redux
- OOP-friendly with classes
- Framework agnostic

## Resources

- [Official Documentation](https://mobx.js.org/)
- [React Integration](https://mobx.js.org/react-integration.html)
- [Best Practices](https://mobx.js.org/best-practices.html)
- [GitHub Repository](https://github.com/mobxjs/mobx)
