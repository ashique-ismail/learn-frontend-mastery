# Signal-Based State Management

## Overview

Signal-based state management represents a fine-grained reactivity paradigm where state is stored in atomic reactive primitives called "signals." Unlike traditional component-based reactivity, signals enable surgical updates that bypass the component tree entirely, updating only the specific DOM nodes that depend on changed values. This approach, popularized by Solid.js and now adopted by Angular, Preact, and others, offers exceptional performance and a simpler mental model for reactive data flow.

## Core Concepts

### What is a Signal?

A signal is a reactive primitive that holds a value and notifies subscribers when that value changes.

```javascript
// Conceptual signal implementation
function createSignal(initialValue) {
  let value = initialValue;
  const subscribers = new Set();
  
  const read = () => {
    // Track which computation is reading this signal
    if (currentListener) {
      subscribers.add(currentListener);
    }
    return value;
  };
  
  const write = (newValue) => {
    value = newValue;
    // Notify all subscribers
    subscribers.forEach(fn => fn());
  };
  
  return [read, write];
}

let currentListener = null;

function createEffect(fn) {
  const execute = () => {
    currentListener = execute;
    fn();
    currentListener = null;
  };
  execute();
}

// Usage
const [count, setCount] = createSignal(0);

createEffect(() => {
  console.log('Count is:', count());
});

setCount(1); // Logs: "Count is: 1"
setCount(2); // Logs: "Count is: 2"
```

### Signal Dependency Graph

```
┌─────────────────────────────────────────────────────────┐
│              Signal Dependency Graph                     │
└─────────────────────────────────────────────────────────┘

    Signal A          Signal B          Signal C
    (firstName)       (lastName)        (age)
         │                 │                │
         └────────┬────────┘                │
                  │                         │
                  ▼                         │
              Computed                      │
              (fullName)                    │
                  │                         │
                  └──────────┬──────────────┘
                             │
                             ▼
                          Effect
                     (Update DOM)
                             │
                             ▼
                    Only affected nodes
                    update - no VDOM diff
```

## Solid.js Signals

### Basic Signals and Effects

```typescript
import { createSignal, createEffect, createMemo } from 'solid-js';

// Counter example
function Counter() {
  const [count, setCount] = createSignal(0);
  const [step, setStep] = createSignal(1);
  
  // Computed signal - automatically tracks dependencies
  const doubled = createMemo(() => count() * 2);
  
  // Effect - runs when dependencies change
  createEffect(() => {
    console.log('Count changed:', count());
  });
  
  const increment = () => setCount(count() + step());
  const decrement = () => setCount(count() - step());
  
  return (
    <div>
      <h2>Count: {count()}</h2>
      <p>Doubled: {doubled()}</p>
      
      <button onClick={increment}>+{step()}</button>
      <button onClick={decrement}>-{step()}</button>
      
      <input 
        type="number" 
        value={step()} 
        onInput={(e) => setStep(Number(e.target.value))}
      />
    </div>
  );
}
```

### Signals with Complex State

```typescript
import { createSignal, createMemo, For } from 'solid-js';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

function TodoApp() {
  const [todos, setTodos] = createSignal<Todo[]>([]);
  const [filter, setFilter] = createSignal<'all' | 'active' | 'completed'>('all');
  const [sortBy, setSortBy] = createSignal<'priority' | 'date'>('date');
  
  // Computed values automatically update
  const filteredTodos = createMemo(() => {
    const list = todos();
    switch (filter()) {
      case 'active':
        return list.filter(t => !t.completed);
      case 'completed':
        return list.filter(t => t.completed);
      default:
        return list;
    }
  });
  
  const sortedTodos = createMemo(() => {
    const list = [...filteredTodos()];
    if (sortBy() === 'priority') {
      const priority = { high: 3, medium: 2, low: 1 };
      return list.sort((a, b) => priority[b.priority] - priority[a.priority]);
    }
    return list;
  });
  
  const stats = createMemo(() => ({
    total: todos().length,
    active: todos().filter(t => !t.completed).length,
    completed: todos().filter(t => t.completed).length,
    highPriority: todos().filter(t => t.priority === 'high' && !t.completed).length
  }));
  
  const addTodo = (text: string, priority: Todo['priority'] = 'medium') => {
    setTodos([...todos(), {
      id: Date.now().toString(),
      text,
      completed: false,
      priority
    }]);
  };
  
  const toggleTodo = (id: string) => {
    setTodos(todos().map(todo => 
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };
  
  const deleteTodo = (id: string) => {
    setTodos(todos().filter(t => t.id !== id));
  };
  
  return (
    <div>
      <h1>Todos</h1>
      
      <div>
        <button onClick={() => setFilter('all')}>All ({stats().total})</button>
        <button onClick={() => setFilter('active')}>Active ({stats().active})</button>
        <button onClick={() => setFilter('completed')}>Completed ({stats().completed})</button>
      </div>
      
      <div>
        <button onClick={() => setSortBy('date')}>Sort by Date</button>
        <button onClick={() => setSortBy('priority')}>Sort by Priority</button>
      </div>
      
      <p>High Priority: {stats().highPriority}</p>
      
      <For each={sortedTodos()}>
        {(todo) => (
          <div>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ 
              'text-decoration': todo.completed ? 'line-through' : 'none' 
            }}>
              {todo.text}
            </span>
            <span>[{todo.priority}]</span>
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </div>
        )}
      </For>
    </div>
  );
}
```

### Solid.js Stores for Nested Reactivity

```typescript
import { createStore } from 'solid-js/store';

interface User {
  id: string;
  name: string;
  email: string;
  settings: {
    theme: 'light' | 'dark';
    notifications: boolean;
    privacy: {
      shareEmail: boolean;
      shareProfile: boolean;
    };
  };
}

function UserProfile() {
  const [user, setUser] = createStore<User>({
    id: '1',
    name: 'John Doe',
    email: 'john@example.com',
    settings: {
      theme: 'light',
      notifications: true,
      privacy: {
        shareEmail: false,
        shareProfile: true
      }
    }
  });
  
  // Fine-grained updates - only affected parts re-render
  const updateName = (name: string) => {
    setUser('name', name);
  };
  
  const toggleNotifications = () => {
    setUser('settings', 'notifications', prev => !prev);
  };
  
  const updatePrivacy = (key: 'shareEmail' | 'shareProfile', value: boolean) => {
    setUser('settings', 'privacy', key, value);
  };
  
  const toggleTheme = () => {
    setUser('settings', 'theme', prev => 
      prev === 'light' ? 'dark' : 'light'
    );
  };
  
  return (
    <div>
      <h2>{user.name}</h2>
      <input
        value={user.name}
        onInput={(e) => updateName(e.currentTarget.value)}
      />
      
      <div>
        <label>
          <input
            type="checkbox"
            checked={user.settings.notifications}
            onChange={toggleNotifications}
          />
          Enable Notifications
        </label>
      </div>
      
      <div>
        <button onClick={toggleTheme}>
          Theme: {user.settings.theme}
        </button>
      </div>
      
      <div>
        <label>
          <input
            type="checkbox"
            checked={user.settings.privacy.shareEmail}
            onChange={(e) => updatePrivacy('shareEmail', e.currentTarget.checked)}
          />
          Share Email
        </label>
      </div>
    </div>
  );
}
```

## Angular Signals

### Basic Angular Signals

```typescript
import { Component, signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <h2>Count: {{ count() }}</h2>
      <p>Doubled: {{ doubled() }}</p>
      <p>Is Even: {{ isEven() }}</p>
      
      <button (click)="increment()">Increment</button>
      <button (click)="decrement()">Decrement</button>
      <button (click)="reset()">Reset</button>
    </div>
  `
})
export class CounterComponent {
  // Signal - writable reactive value
  count = signal(0);
  
  // Computed signal - derived value
  doubled = computed(() => this.count() * 2);
  isEven = computed(() => this.count() % 2 === 0);
  
  constructor() {
    // Effect - runs when dependencies change
    effect(() => {
      console.log('Count is:', this.count());
      
      if (this.count() > 10) {
        console.log('Count is high!');
      }
    });
  }
  
  increment() {
    this.count.update(value => value + 1);
  }
  
  decrement() {
    this.count.update(value => value - 1);
  }
  
  reset() {
    this.count.set(0);
  }
}
```

### Angular Signals with Services

```typescript
// user.service.ts
import { Injectable, signal, computed } from '@angular/core';

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

@Injectable({ providedIn: 'root' })
export class UserService {
  private usersSignal = signal<User[]>([]);
  private selectedIdSignal = signal<string | null>(null);
  
  // Public read-only signals
  users = this.usersSignal.asReadonly();
  selectedId = this.selectedIdSignal.asReadonly();
  
  // Computed values
  selectedUser = computed(() => {
    const id = this.selectedIdSignal();
    return this.usersSignal().find(u => u.id === id) ?? null;
  });
  
  adminUsers = computed(() => 
    this.usersSignal().filter(u => u.role === 'admin')
  );
  
  userCount = computed(() => this.usersSignal().length);
  
  // Actions
  addUser(user: User) {
    this.usersSignal.update(users => [...users, user]);
  }
  
  updateUser(id: string, updates: Partial<User>) {
    this.usersSignal.update(users =>
      users.map(u => u.id === id ? { ...u, ...updates } : u)
    );
  }
  
  deleteUser(id: string) {
    this.usersSignal.update(users => users.filter(u => u.id !== id));
  }
  
  selectUser(id: string) {
    this.selectedIdSignal.set(id);
  }
}

// user-list.component.ts
@Component({
  selector: 'app-user-list',
  template: `
    <div>
      <h2>Users ({{ userService.userCount() }})</h2>
      <p>Admins: {{ userService.adminUsers().length }}</p>
      
      <div *ngFor="let user of userService.users()">
        <span>{{ user.name }}</span>
        <span [class.selected]="user.id === userService.selectedId()">
          {{ user.role }}
        </span>
        <button (click)="userService.selectUser(user.id)">Select</button>
        <button (click)="userService.deleteUser(user.id)">Delete</button>
      </div>
      
      <div *ngIf="userService.selectedUser() as user">
        <h3>Selected: {{ user.name }}</h3>
        <p>{{ user.email }}</p>
      </div>
    </div>
  `
})
export class UserListComponent {
  constructor(public userService: UserService) {}
}
```

### Angular Signals with HTTP

```typescript
import { Injectable, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private productsSignal = signal<Product[]>([]);
  private loadingSignal = signal(false);
  private errorSignal = signal<string | null>(null);
  
  products = this.productsSignal.asReadonly();
  loading = this.loadingSignal.asReadonly();
  error = this.errorSignal.asReadonly();
  
  // Computed values
  inStockProducts = computed(() =>
    this.productsSignal().filter(p => p.inStock)
  );
  
  totalValue = computed(() =>
    this.productsSignal().reduce((sum, p) => sum + p.price, 0)
  );
  
  constructor(private http: HttpClient) {}
  
  async loadProducts() {
    this.loadingSignal.set(true);
    this.errorSignal.set(null);
    
    try {
      const products = await this.http
        .get<Product[]>('/api/products')
        .toPromise();
      
      this.productsSignal.set(products || []);
    } catch (error) {
      this.errorSignal.set(error.message);
    } finally {
      this.loadingSignal.set(false);
    }
  }
  
  async updateProduct(id: string, updates: Partial<Product>) {
    // Optimistic update
    const original = this.productsSignal();
    this.productsSignal.update(products =>
      products.map(p => p.id === id ? { ...p, ...updates } : p)
    );
    
    try {
      await this.http
        .patch(`/api/products/${id}`, updates)
        .toPromise();
    } catch (error) {
      // Rollback on error
      this.productsSignal.set(original);
      this.errorSignal.set(error.message);
    }
  }
}

@Component({
  selector: 'app-products',
  template: `
    <div>
      <button (click)="productService.loadProducts()">
        Load Products
      </button>
      
      <div *ngIf="productService.loading()">Loading...</div>
      <div *ngIf="productService.error() as error">Error: {{ error }}</div>
      
      <div *ngIf="!productService.loading()">
        <h3>In Stock: {{ productService.inStockProducts().length }}</h3>
        <h3>Total Value: ${{ productService.totalValue() }}</h3>
        
        <div *ngFor="let product of productService.products()">
          <span>{{ product.name }}</span>
          <span>${{ product.price }}</span>
          <span [class.in-stock]="product.inStock">
            {{ product.inStock ? 'In Stock' : 'Out of Stock' }}
          </span>
        </div>
      </div>
    </div>
  `
})
export class ProductsComponent {
  constructor(public productService: ProductService) {
    this.productService.loadProducts();
  }
}
```

## Preact Signals

### Basic Preact Signals

```typescript
import { signal, computed, effect } from '@preact/signals';

// Global signals
const count = signal(0);
const multiplier = signal(2);

// Computed signal
const result = computed(() => count.value * multiplier.value);

// Effect
effect(() => {
  console.log('Result:', result.value);
});

function Counter() {
  return (
    <div>
      <h2>Count: {count}</h2>
      <h2>Result: {result}</h2>
      
      <button onClick={() => count.value++}>Increment</button>
      <button onClick={() => multiplier.value++}>Increase Multiplier</button>
    </div>
  );
}
```

### Preact Signals in Components

```typescript
import { signal, computed } from '@preact/signals';
import { useSignal, useComputed } from '@preact/signals';

interface Task {
  id: string;
  title: string;
  completed: boolean;
  tags: string[];
}

// Global store
const tasks = signal<Task[]>([]);
const searchQuery = signal('');
const selectedTag = signal<string | null>(null);

// Global computed values
const filteredTasks = computed(() => {
  let result = tasks.value;
  
  if (searchQuery.value) {
    result = result.filter(t =>
      t.title.toLowerCase().includes(searchQuery.value.toLowerCase())
    );
  }
  
  if (selectedTag.value) {
    result = result.filter(t => t.tags.includes(selectedTag.value!));
  }
  
  return result;
});

const allTags = computed(() => {
  const tagSet = new Set<string>();
  tasks.value.forEach(task => {
    task.tags.forEach(tag => tagSet.add(tag));
  });
  return Array.from(tagSet);
});

// Actions
export const taskActions = {
  addTask(title: string, tags: string[] = []) {
    tasks.value = [...tasks.value, {
      id: Date.now().toString(),
      title,
      completed: false,
      tags
    }];
  },
  
  toggleTask(id: string) {
    tasks.value = tasks.value.map(task =>
      task.id === id ? { ...task, completed: !task.completed } : task
    );
  },
  
  deleteTask(id: string) {
    tasks.value = tasks.value.filter(t => t.id !== id);
  }
};

function TaskList() {
  return (
    <div>
      <input
        type="text"
        placeholder="Search..."
        value={searchQuery}
        onInput={(e) => searchQuery.value = e.currentTarget.value}
      />
      
      <div>
        <button onClick={() => selectedTag.value = null}>All</button>
        {allTags.value.map(tag => (
          <button
            key={tag}
            onClick={() => selectedTag.value = tag}
            class={selectedTag.value === tag ? 'active' : ''}
          >
            {tag}
          </button>
        ))}
      </div>
      
      <div>
        {filteredTasks.value.map(task => (
          <div key={task.id}>
            <input
              type="checkbox"
              checked={task.completed}
              onChange={() => taskActions.toggleTask(task.id)}
            />
            <span>{task.title}</span>
            <span>{task.tags.join(', ')}</span>
            <button onClick={() => taskActions.deleteTask(task.id)}>
              Delete
            </button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Advanced Signal Patterns

### Batched Updates

```typescript
import { batch, signal } from '@preact/signals';

const firstName = signal('John');
const lastName = signal('Doe');
const age = signal(30);

// Without batching - 3 effect runs
firstName.value = 'Jane';
lastName.value = 'Smith';
age.value = 25;

// With batching - 1 effect run
batch(() => {
  firstName.value = 'Jane';
  lastName.value = 'Smith';
  age.value = 25;
});
```

### Signal Selectors

```typescript
import { signal, computed } from 'solid-js';

interface State {
  users: User[];
  products: Product[];
  orders: Order[];
}

const state = signal<State>({
  users: [],
  products: [],
  orders: []
});

// Create selector function
function createSelector<T, R>(
  source: () => T,
  selector: (state: T) => R,
  equals: (a: R, b: R) => boolean = (a, b) => a === b
): () => R {
  return createMemo(() => selector(source()), undefined, { equals });
}

// Usage
const selectUsers = createSelector(
  state,
  (s) => s.users,
  (a, b) => a.length === b.length && a.every((user, i) => user === b[i])
);

const selectUserById = (id: string) =>
  createMemo(() => selectUsers().find(u => u.id === id));
```

### Signal-based State Machine

```typescript
import { signal, computed } from '@preact/signals';

type TrafficLightState = 'red' | 'yellow' | 'green';

const currentState = signal<TrafficLightState>('red');

const canTransition = computed(() => ({
  toYellow: currentState.value === 'red',
  toGreen: currentState.value === 'yellow',
  toRed: currentState.value === 'green'
}));

const transitions = {
  next() {
    switch (currentState.value) {
      case 'red':
        currentState.value = 'yellow';
        break;
      case 'yellow':
        currentState.value = 'green';
        break;
      case 'green':
        currentState.value = 'red';
        break;
    }
  },
  
  reset() {
    currentState.value = 'red';
  }
};

function TrafficLight() {
  return (
    <div>
      <div class={`light ${currentState.value}`}></div>
      <button 
        onClick={() => transitions.next()}
        disabled={!canTransition.value}
      >
        Next
      </button>
    </div>
  );
}
```

### Async Signals

```typescript
import { signal, computed } from '@preact/signals';

interface AsyncState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function createAsyncSignal<T>(fetchFn: () => Promise<T>) {
  const state = signal<AsyncState<T>>({
    data: null,
    loading: false,
    error: null
  });
  
  const execute = async () => {
    state.value = { data: null, loading: true, error: null };
    
    try {
      const data = await fetchFn();
      state.value = { data, loading: false, error: null };
    } catch (error) {
      state.value = { data: null, loading: false, error: error as Error };
    }
  };
  
  const reset = () => {
    state.value = { data: null, loading: false, error: null };
  };
  
  return {
    state,
    execute,
    reset,
    data: computed(() => state.value.data),
    loading: computed(() => state.value.loading),
    error: computed(() => state.value.error)
  };
}

// Usage
const userAsync = createAsyncSignal(() =>
  fetch('/api/user').then(r => r.json())
);

function UserProfile() {
  useEffect(() => {
    userAsync.execute();
  }, []);
  
  return (
    <div>
      {userAsync.loading.value && <div>Loading...</div>}
      {userAsync.error.value && <div>Error: {userAsync.error.value.message}</div>}
      {userAsync.data.value && <div>User: {userAsync.data.value.name}</div>}
    </div>
  );
}
```

## Common Mistakes

### 1. Forgetting to Call Signal Functions

```typescript
// ❌ Wrong - not calling the signal
const [count, setCount] = createSignal(0);
console.log(count); // Logs the function, not the value

// ✅ Correct - call the signal to get value
console.log(count()); // Logs: 0
```

### 2. Creating Signals in Render

```typescript
// ❌ Wrong - creates new signal every render
function BadComponent() {
  const [count, setCount] = createSignal(0);
  return <div>{count()}</div>;
}

// ✅ Correct - signal created once
const [count, setCount] = createSignal(0);
function GoodComponent() {
  return <div>{count()}</div>;
}
```

### 3. Not Using Computed for Derived Values

```typescript
// ❌ Wrong - recomputes every time
function BadComponent() {
  const [count, setCount] = createSignal(0);
  const doubled = () => count() * 2; // Not memoized
  
  return <div>{doubled()}</div>;
}

// ✅ Correct - memoized computed value
function GoodComponent() {
  const [count, setCount] = createSignal(0);
  const doubled = createMemo(() => count() * 2);
  
  return <div>{doubled()}</div>;
}
```

### 4. Mutating Signal Values Directly

```typescript
// ❌ Wrong - mutating the array
const [items, setItems] = createSignal([]);
items().push(newItem); // Doesn't trigger updates

// ✅ Correct - create new array
setItems([...items(), newItem]);
```

### 5. Effect Dependencies Not Tracked

```typescript
// ❌ Wrong - dependency not tracked
const [count, setCount] = createSignal(0);
const value = count();

createEffect(() => {
  console.log(value); // Won't update when count changes
});

// ✅ Correct - read signal inside effect
createEffect(() => {
  console.log(count()); // Tracks dependency
});
```

## Best Practices

### 1. Use Signals for Atomic State

```typescript
// Good - separate signals for independent state
const firstName = signal('John');
const lastName = signal('Doe');
const age = signal(30);

// Computed values combine them
const fullName = computed(() => `${firstName.value} ${lastName.value}`);
```

### 2. Prefer Computed Over Effects

```typescript
// ❌ Avoid - using effect for derived state
const [count, setCount] = createSignal(0);
const [doubled, setDoubled] = createSignal(0);

createEffect(() => {
  setDoubled(count() * 2);
});

// ✅ Better - use computed
const doubled = createMemo(() => count() * 2);
```

### 3. Co-locate Related Signals

```typescript
// Good - related signals grouped together
function createUserStore() {
  const [user, setUser] = createSignal(null);
  const [loading, setLoading] = createSignal(false);
  const [error, setError] = createSignal(null);
  
  const isAuthenticated = createMemo(() => user() !== null);
  
  return {
    user,
    loading,
    error,
    isAuthenticated,
    setUser,
    setLoading,
    setError
  };
}
```

### 4. Use TypeScript for Signal Types

```typescript
interface User {
  id: string;
  name: string;
}

const user = signal<User | null>(null);
const users = signal<User[]>([]);
```

## When to Use Signal-Based State

### Use When:
- Performance is critical (fine-grained updates)
- You want minimal re-renders
- State changes are frequent but localized
- You prefer reactive programming models
- Building data-heavy UIs (tables, charts)

### Avoid When:
- Team unfamiliar with reactive programming
- Simple applications with infrequent updates
- You need mature ecosystem (Redux DevTools, etc.)
- Debugging reactive flows is challenging for your team
- Integration with existing non-signal libraries

## Interview Questions

### Q1: How do signals differ from useState?
**Answer**: Signals provide fine-grained reactivity - only DOM nodes depending on changed signals update, bypassing component re-renders entirely. useState triggers component re-renders. Signals track dependencies automatically through getter calls. Signals can be global without context. Performance scales better with signals for complex UIs.

### Q2: What is the purpose of computed/memo in signals?
**Answer**: Computed signals cache derived values and only recompute when dependencies change. They prevent unnecessary recalculations and track their own dependencies automatically. Without computed, derived values would recalculate on every access, even if inputs haven't changed.

### Q3: How do signals achieve fine-grained reactivity?
**Answer**: Signals track which effects/computations read them during execution by maintaining a dependency graph. When a signal updates, only the specific effects that depend on it run, updating only the affected DOM nodes. This bypasses virtual DOM diffing and component re-renders entirely.

### Q4: What are the trade-offs of signal-based state?
**Answer**: Pros: exceptional performance, minimal re-renders, simple mental model, automatic dependency tracking. Cons: different paradigm from React, debugging reactive flows can be challenging, smaller ecosystem, requires understanding of reactive programming, function call syntax can be confusing initially.

### Q5: When should you use signals vs traditional state?
**Answer**: Use signals for performance-critical apps with frequent updates, data-heavy UIs, and when you want fine-grained control. Use traditional state for simpler apps, when team familiarity matters, or when you need mature tooling. Signals excel at scale but have a learning curve.

## Key Takeaways

1. Signals enable surgical DOM updates without virtual DOM diffing
2. Automatic dependency tracking eliminates manual subscriptions
3. Computed signals cache derived values efficiently
4. Effects run only when their signal dependencies change
5. Signals can be global without React context overhead
6. Fine-grained reactivity scales better for complex UIs
7. Function call syntax (signal()) takes getting used to
8. Batch updates when changing multiple signals together
9. Prefer computed over effects for derived state
10. TypeScript integration provides excellent type safety

## Resources

- [Solid.js Documentation](https://www.solidjs.com/docs/latest)
- [Angular Signals Guide](https://angular.io/guide/signals)
- [Preact Signals](https://preactjs.com/guide/v10/signals/)
- [Signals vs React State](https://dev.to/this-is-learning/the-evolution-of-signals-in-javascript-8ob)
- [Fine-Grained Reactivity](https://dev.to/ryansolid/a-hands-on-introduction-to-fine-grained-reactivity-3ndf)
- [Signals Proposal](https://github.com/tc39/proposal-signals)
