# Proxy-Based State Management

## The Idea

**In plain English:** Proxy-based state management is a way for a program to secretly watch an object and automatically notice whenever something inside it gets read or changed, then instantly update the screen without you having to tell it to. Think of the "object" as a box of data your app uses to decide what to show.

**Real-world analogy:** Imagine a hotel concierge who stands between you and the guest registry book. Every time a guest looks up a room number, the concierge notes who checked what. Every time a guest's details change, the concierge immediately calls every staff member who previously looked up that guest to give them the update.

- The concierge = the Proxy (intercepts all access and changes)
- The guest registry book = the actual state object (holds the real data)
- The staff members who checked a guest's details = the components that read that piece of state (they get notified on change)

---

## Overview

Proxy-based state management leverages JavaScript's Proxy API to intercept and track object property access, enabling automatic reactivity and state tracking without requiring explicit subscriptions or immutable updates. Libraries like Valtio and MobX use proxies to provide a mutable API while maintaining immutable state underneath, offering a developer experience similar to working with plain JavaScript objects.

## Core Concepts

### JavaScript Proxy Fundamentals

The Proxy object wraps another object and intercepts operations performed on it.

```javascript
// Basic proxy example
const target = { count: 0, name: 'John' };

const handler = {
  get(target, property, receiver) {
    console.log(`Getting ${property}`);
    return Reflect.get(target, property, receiver);
  },
  set(target, property, value, receiver) {
    console.log(`Setting ${property} to ${value}`);
    return Reflect.set(target, property, value, receiver);
  }
};

const proxy = new Proxy(target, handler);

proxy.count; // Logs: "Getting count"
proxy.count = 1; // Logs: "Setting count to 1"
```

### Automatic Dependency Tracking

```javascript
// Conceptual implementation of reactive proxy
class ReactiveProxy {
  constructor(target) {
    this.subscribers = new Map();
    this.proxy = new Proxy(target, {
      get: (target, property) => {
        this.track(property);
        return target[property];
      },
      set: (target, property, value) => {
        target[property] = value;
        this.trigger(property);
        return true;
      }
    });
  }

  track(property) {
    if (ReactiveProxy.currentEffect) {
      if (!this.subscribers.has(property)) {
        this.subscribers.set(property, new Set());
      }
      this.subscribers.get(property).add(ReactiveProxy.currentEffect);
    }
  }

  trigger(property) {
    const effects = this.subscribers.get(property);
    if (effects) {
      effects.forEach(effect => effect());
    }
  }

  static currentEffect = null;

  static effect(fn) {
    ReactiveProxy.currentEffect = fn;
    fn();
    ReactiveProxy.currentEffect = null;
  }
}

// Usage
const state = new ReactiveProxy({ count: 0 });

ReactiveProxy.effect(() => {
  console.log('Count is:', state.proxy.count);
});

state.proxy.count = 1; // Logs: "Count is: 1"
state.proxy.count = 2; // Logs: "Count is: 2"
```

## Valtio - Proxy State Made Simple

### Basic Valtio Store (React)

```typescript
// store/userStore.ts
import { proxy, useSnapshot } from 'valtio';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserState {
  users: User[];
  selectedUserId: string | null;
  loading: boolean;
}

export const userStore = proxy<UserState>({
  users: [],
  selectedUserId: null,
  loading: false
});

// Actions
export const userActions = {
  addUser(user: User) {
    userStore.users.push(user);
  },
  
  selectUser(id: string) {
    userStore.selectedUserId = id;
  },
  
  updateUser(id: string, updates: Partial<User>) {
    const user = userStore.users.find(u => u.id === id);
    if (user) {
      Object.assign(user, updates);
    }
  },
  
  deleteUser(id: string) {
    const index = userStore.users.findIndex(u => u.id === id);
    if (index !== -1) {
      userStore.users.splice(index, 1);
    }
  }
};

// Component usage
import React from 'react';
import { useSnapshot } from 'valtio';
import { userStore, userActions } from './store/userStore';

function UserList() {
  const snap = useSnapshot(userStore);
  
  return (
    <div>
      <h2>Users ({snap.users.length})</h2>
      {snap.users.map(user => (
        <div key={user.id}>
          <span>{user.name}</span>
          <button onClick={() => userActions.selectUser(user.id)}>
            Select
          </button>
        </div>
      ))}
      <button 
        onClick={() => userActions.addUser({
          id: Date.now().toString(),
          name: 'New User',
          email: 'new@example.com'
        })}
      >
        Add User
      </button>
    </div>
  );
}
```

### Valtio with Derived State

```typescript
import { proxy, useSnapshot, derive } from 'valtio/utils';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
}

const todoStore = proxy<TodoState>({
  todos: [],
  filter: 'all'
});

// Derived state - computed automatically
const derivedTodoStore = derive({
  filteredTodos: (get) => {
    const state = get(todoStore);
    switch (state.filter) {
      case 'active':
        return state.todos.filter(t => !t.completed);
      case 'completed':
        return state.todos.filter(t => t.completed);
      default:
        return state.todos;
    }
  },
  
  highPriorityCount: (get) => {
    const state = get(todoStore);
    return state.todos.filter(t => t.priority === 'high' && !t.completed).length;
  },
  
  completionRate: (get) => {
    const state = get(todoStore);
    if (state.todos.length === 0) return 0;
    const completed = state.todos.filter(t => t.completed).length;
    return Math.round((completed / state.todos.length) * 100);
  }
});

function TodoStats() {
  const snap = useSnapshot(derivedTodoStore);
  
  return (
    <div>
      <p>High Priority: {snap.highPriorityCount}</p>
      <p>Completion: {snap.completionRate}%</p>
    </div>
  );
}
```

### Valtio with Async Actions

```typescript
import { proxy, ref } from 'valtio';

interface Product {
  id: string;
  name: string;
  price: number;
}

interface ProductState {
  products: Product[];
  loading: boolean;
  error: Error | null;
  abortController: AbortController | null;
}

const productStore = proxy<ProductState>({
  products: [],
  loading: false,
  error: null,
  abortController: null
});

export const productActions = {
  async fetchProducts() {
    // Cancel previous request
    if (productStore.abortController) {
      productStore.abortController.abort();
    }
    
    const controller = new AbortController();
    productStore.abortController = ref(controller);
    productStore.loading = true;
    productStore.error = null;
    
    try {
      const response = await fetch('/api/products', {
        signal: controller.signal
      });
      const data = await response.json();
      productStore.products = data;
    } catch (error) {
      if (error.name !== 'AbortError') {
        productStore.error = error as Error;
      }
    } finally {
      productStore.loading = false;
      productStore.abortController = null;
    }
  },
  
  async updateProduct(id: string, updates: Partial<Product>) {
    const product = productStore.products.find(p => p.id === id);
    if (!product) return;
    
    // Optimistic update
    const original = { ...product };
    Object.assign(product, updates);
    
    try {
      await fetch(`/api/products/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(updates),
        headers: { 'Content-Type': 'application/json' }
      });
    } catch (error) {
      // Rollback on error
      Object.assign(product, original);
      productStore.error = error as Error;
    }
  }
};
```

## MobX - Observable State Management

### MobX Observable Store

```typescript
import { makeObservable, observable, action, computed } from 'mobx';

class ShoppingCartStore {
  items: Array<{ id: string; name: string; price: number; quantity: number }> = [];
  tax = 0.08;
  
  constructor() {
    makeObservable(this, {
      items: observable,
      tax: observable,
      addItem: action,
      removeItem: action,
      updateQuantity: action,
      clear: action,
      subtotal: computed,
      taxAmount: computed,
      total: computed,
      itemCount: computed
    });
  }
  
  addItem(id: string, name: string, price: number) {
    const existing = this.items.find(item => item.id === id);
    if (existing) {
      existing.quantity++;
    } else {
      this.items.push({ id, name, price, quantity: 1 });
    }
  }
  
  removeItem(id: string) {
    const index = this.items.findIndex(item => item.id === id);
    if (index !== -1) {
      this.items.splice(index, 1);
    }
  }
  
  updateQuantity(id: string, quantity: number) {
    const item = this.items.find(i => i.id === id);
    if (item) {
      item.quantity = Math.max(0, quantity);
    }
  }
  
  clear() {
    this.items = [];
  }
  
  get subtotal() {
    return this.items.reduce((sum, item) => 
      sum + (item.price * item.quantity), 0
    );
  }
  
  get taxAmount() {
    return this.subtotal * this.tax;
  }
  
  get total() {
    return this.subtotal + this.taxAmount;
  }
  
  get itemCount() {
    return this.items.reduce((sum, item) => sum + item.quantity, 0);
  }
}

export const cartStore = new ShoppingCartStore();
```

### MobX with React

```typescript
import React from 'react';
import { observer } from 'mobx-react-lite';
import { cartStore } from './stores/cartStore';

const CartSummary = observer(() => {
  return (
    <div>
      <h2>Cart Summary</h2>
      <p>Items: {cartStore.itemCount}</p>
      <p>Subtotal: ${cartStore.subtotal.toFixed(2)}</p>
      <p>Tax: ${cartStore.taxAmount.toFixed(2)}</p>
      <h3>Total: ${cartStore.total.toFixed(2)}</h3>
      <button onClick={() => cartStore.clear()}>Clear Cart</button>
    </div>
  );
});

const CartItem = observer(({ id }: { id: string }) => {
  const item = cartStore.items.find(i => i.id === id);
  if (!item) return null;
  
  return (
    <div>
      <span>{item.name}</span>
      <span>${item.price}</span>
      <input
        type="number"
        value={item.quantity}
        onChange={(e) => cartStore.updateQuantity(id, parseInt(e.target.value))}
      />
      <button onClick={() => cartStore.removeItem(id)}>Remove</button>
    </div>
  );
});
```

### MobX with Angular

```typescript
// cart.store.ts
import { Injectable } from '@angular/core';
import { makeObservable, observable, action, computed } from 'mobx';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

@Injectable({ providedIn: 'root' })
export class CartStore {
  items: CartItem[] = [];
  tax = 0.08;

  constructor() {
    makeObservable(this, {
      items: observable,
      tax: observable,
      addItem: action,
      removeItem: action,
      subtotal: computed,
      total: computed
    });
  }

  addItem(id: string, name: string, price: number) {
    const existing = this.items.find(item => item.id === id);
    if (existing) {
      existing.quantity++;
    } else {
      this.items.push({ id, name, price, quantity: 1 });
    }
  }

  removeItem(id: string) {
    const index = this.items.findIndex(item => item.id === id);
    if (index !== -1) {
      this.items.splice(index, 1);
    }
  }

  get subtotal() {
    return this.items.reduce((sum, item) => 
      sum + (item.price * item.quantity), 0
    );
  }

  get total() {
    return this.subtotal * (1 + this.tax);
  }
}

// cart.component.ts
import { Component } from '@angular/core';
import { CartStore } from './cart.store';
import { autorun } from 'mobx';

@Component({
  selector: 'app-cart',
  template: `
    <div>
      <h2>Shopping Cart</h2>
      <div *ngFor="let item of store.items">
        <span>{{ item.name }}</span>
        <span>{{ item.quantity }}</span>
        <button (click)="store.removeItem(item.id)">Remove</button>
      </div>
      <p>Total: {{ store.total | currency }}</p>
    </div>
  `
})
export class CartComponent {
  constructor(public store: CartStore) {
    // Auto-update component when store changes
    autorun(() => {
      console.log('Cart total:', this.store.total);
    });
  }
}
```

## Advanced Patterns

### Nested Proxy State

```typescript
import { proxy, useSnapshot } from 'valtio';

interface Address {
  street: string;
  city: string;
  country: string;
}

interface Company {
  name: string;
  addresses: Address[];
}

interface AppState {
  companies: Company[];
}

const appStore = proxy<AppState>({
  companies: []
});

// Nested mutations work automatically
export const companyActions = {
  addCompany(name: string) {
    appStore.companies.push({
      name,
      addresses: []
    });
  },
  
  addAddress(companyIndex: number, address: Address) {
    appStore.companies[companyIndex].addresses.push(address);
  },
  
  updateAddress(companyIndex: number, addressIndex: number, updates: Partial<Address>) {
    const address = appStore.companies[companyIndex].addresses[addressIndex];
    Object.assign(address, updates);
  }
};

function CompanyAddresses({ companyIndex }: { companyIndex: number }) {
  const snap = useSnapshot(appStore);
  const company = snap.companies[companyIndex];
  
  return (
    <div>
      <h3>{company.name} - Addresses</h3>
      {company.addresses.map((address, index) => (
        <div key={index}>
          <p>{address.street}, {address.city}, {address.country}</p>
          <button 
            onClick={() => companyActions.updateAddress(
              companyIndex, 
              index, 
              { city: 'New City' }
            )}
          >
            Update City
          </button>
        </div>
      ))}
    </div>
  );
}
```

### Proxy with Middleware

```typescript
import { proxy, subscribe } from 'valtio';

interface LogEntry {
  timestamp: number;
  type: 'get' | 'set';
  path: string[];
  value?: any;
}

function createLoggedProxy<T extends object>(target: T): T {
  const logs: LogEntry[] = [];
  
  const handler: ProxyHandler<T> = {
    get(target, property, receiver) {
      logs.push({
        timestamp: Date.now(),
        type: 'get',
        path: [String(property)]
      });
      return Reflect.get(target, property, receiver);
    },
    
    set(target, property, value, receiver) {
      logs.push({
        timestamp: Date.now(),
        type: 'set',
        path: [String(property)],
        value
      });
      return Reflect.set(target, property, value, receiver);
    }
  };
  
  const proxied = new Proxy(target, handler);
  
  // Attach log retrieval
  Object.defineProperty(proxied, '__logs__', {
    get() { return logs; },
    enumerable: false
  });
  
  return proxied;
}

// Valtio with history tracking
function createHistoryProxy<T extends object>(initialState: T) {
  const state = proxy(initialState);
  const history: T[] = [JSON.parse(JSON.stringify(initialState))];
  let historyIndex = 0;
  
  subscribe(state, () => {
    // Remove future history when new change is made
    history.splice(historyIndex + 1);
    history.push(JSON.parse(JSON.stringify(state)));
    historyIndex = history.length - 1;
  });
  
  return {
    state,
    undo() {
      if (historyIndex > 0) {
        historyIndex--;
        Object.assign(state, history[historyIndex]);
      }
    },
    redo() {
      if (historyIndex < history.length - 1) {
        historyIndex++;
        Object.assign(state, history[historyIndex]);
      }
    },
    canUndo: () => historyIndex > 0,
    canRedo: () => historyIndex < history.length - 1
  };
}

// Usage
interface EditorState {
  content: string;
  cursorPosition: number;
}

const editor = createHistoryProxy<EditorState>({
  content: '',
  cursorPosition: 0
});

function TextEditor() {
  const snap = useSnapshot(editor.state);
  
  return (
    <div>
      <textarea
        value={snap.content}
        onChange={(e) => {
          editor.state.content = e.target.value;
        }}
      />
      <button 
        onClick={() => editor.undo()} 
        disabled={!editor.canUndo()}
      >
        Undo
      </button>
      <button 
        onClick={() => editor.redo()} 
        disabled={!editor.canRedo()}
      >
        Redo
      </button>
    </div>
  );
}
```

### Proxy Performance Optimization

```typescript
import { proxy, useSnapshot, ref } from 'valtio';

interface LargeDataItem {
  id: string;
  data: string;
  metadata: object;
}

interface AppState {
  // Use ref() for non-reactive data
  config: object;
  
  // Large arrays benefit from virtualization
  items: LargeDataItem[];
  
  // Frequently changing data
  mousePosition: { x: number; y: number };
}

const appStore = proxy<AppState>({
  config: ref({ /* large config object */ }),
  items: [],
  mousePosition: { x: 0, y: 0 }
});

// Batch updates for better performance
export function batchAddItems(newItems: LargeDataItem[]) {
  // Single mutation that triggers one re-render
  appStore.items.push(...newItems);
}

// Selective subscription
function OptimizedComponent() {
  // Only subscribe to specific properties
  const items = useSnapshot(appStore, { sync: true }).items;
  
  return (
    <div>
      {items.slice(0, 100).map(item => (
        <div key={item.id}>{item.data}</div>
      ))}
    </div>
  );
}

// Debounced updates for high-frequency changes
import { debounce } from 'lodash';

const updateMousePosition = debounce((x: number, y: number) => {
  appStore.mousePosition.x = x;
  appStore.mousePosition.y = y;
}, 16); // ~60fps

function MouseTracker() {
  React.useEffect(() => {
    const handler = (e: MouseEvent) => {
      updateMousePosition(e.clientX, e.clientY);
    };
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  
  const position = useSnapshot(appStore).mousePosition;
  return <div>Mouse: ({position.x}, {position.y})</div>;
}
```

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Proxy-Based State Flow                    │
└─────────────────────────────────────────────────────────────┘

Component A          Component B          Component C
     │                    │                    │
     │ useSnapshot()      │ useSnapshot()      │ useSnapshot()
     │                    │                    │
     └────────────────────┼────────────────────┘
                          │
                          ▼
            ┌─────────────────────────┐
            │   Proxy State Store     │
            │  ┌─────────────────┐   │
            │  │  Actual State   │   │
            │  │  { count: 0 }   │   │
            │  └─────────────────┘   │
            │           ▲             │
            │           │             │
            │  ┌────────┴────────┐   │
            │  │  Proxy Handler  │   │
            │  │   get() / set() │   │
            │  └─────────────────┘   │
            └─────────────────────────┘
                     │     ▲
            Mutation │     │ Read
                     ▼     │
            ┌──────────────────────┐
            │  Dependency Tracker  │
            │  ┌────────────────┐  │
            │  │  Component A   │  │
            │  │  Component B   │  │
            │  │  Component C   │  │
            │  └────────────────┘  │
            └──────────────────────┘
                     │
                     │ Trigger Updates
                     ▼
            Only re-render components
            that access changed properties
```

## Common Mistakes

### 1. Destructuring Snapshots Too Early

```typescript
// ❌ Wrong - loses reactivity
function BadComponent() {
  const { count, name } = useSnapshot(store);
  // count and name are now plain values, won't update
  return <div>{count}</div>;
}

// ✅ Correct - maintain snapshot reference
function GoodComponent() {
  const snap = useSnapshot(store);
  return <div>{snap.count}</div>;
}
```

### 2. Mutating Snapshots

```typescript
// ❌ Wrong - snapshots are read-only
function BadComponent() {
  const snap = useSnapshot(store);
  snap.count++; // Error! Snapshot is immutable
  return <div>{snap.count}</div>;
}

// ✅ Correct - mutate the store, not snapshot
function GoodComponent() {
  const snap = useSnapshot(store);
  return (
    <button onClick={() => store.count++}>
      {snap.count}
    </button>
  );
}
```

### 3. Not Using ref() for Non-Reactive Data

```typescript
// ❌ Wrong - makes everything reactive
const store = proxy({
  largeConfig: { /* huge object */ },
  data: []
});

// ✅ Correct - exclude from reactivity
const store = proxy({
  largeConfig: ref({ /* huge object */ }),
  data: []
});
```

### 4. Forgetting observer() in MobX

```typescript
// ❌ Wrong - component won't update
function CartTotal() {
  return <div>{cartStore.total}</div>;
}

// ✅ Correct - wrap with observer
const CartTotal = observer(() => {
  return <div>{cartStore.total}</div>;
});
```

### 5. Proxy Limitations with Built-in Objects

```typescript
// ❌ Problematic - Maps/Sets need special handling
const store = proxy({
  items: new Set() // Won't track properly
});

// ✅ Better - use arrays or Valtio's ref
const store = proxy({
  items: [] // Use arrays
});

// Or with ref for Sets/Maps
const store = proxy({
  itemSet: ref(new Set())
});
```

## Best Practices

### 1. Organize Store by Domain

```typescript
// stores/index.ts
import { proxy } from 'valtio';

export const userStore = proxy({
  currentUser: null,
  preferences: {}
});

export const productsStore = proxy({
  items: [],
  filters: {}
});

export const cartStore = proxy({
  items: [],
  checkout: {}
});
```

### 2. Encapsulate Actions

```typescript
// Don't expose raw store
const _store = proxy({ count: 0 });

// Export actions and read-only access
export const counterActions = {
  increment: () => _store.count++,
  decrement: () => _store.count--,
  reset: () => _store.count = 0
};

export const useCounterStore = () => useSnapshot(_store);
```

### 3. Use Computed Values

```typescript
import { derive } from 'valtio/utils';

const store = proxy({
  items: [],
  filter: 'all'
});

export const derived = derive({
  filteredItems: (get) => {
    const { items, filter } = get(store);
    return filter === 'all' 
      ? items 
      : items.filter(i => i.status === filter);
  }
});
```

### 4. TypeScript Best Practices

```typescript
// Strong typing for store and actions
interface UserStore {
  users: User[];
  selectedId: string | null;
}

const userStore = proxy<UserStore>({
  users: [],
  selectedId: null
});

// Type-safe actions
export const userActions: {
  [K in keyof typeof userActionsImpl]: typeof userActionsImpl[K]
} = {
  addUser: (user: User) => {
    userStore.users.push(user);
  }
};
```

## When to Use Proxy-Based State

### Use When:
- Building complex applications with nested state
- You prefer mutable-style APIs
- You need automatic dependency tracking
- You want minimal boilerplate
- Performance is critical (fine-grained updates)

### Avoid When:
- You need Redux DevTools integration (limited support)
- Your team prefers immutable patterns
- You're working with legacy browsers (Proxy support)
- You need time-travel debugging
- State changes need to be serializable for auditing

## Interview Questions

### Q1: How do JavaScript Proxies enable automatic reactivity?
**Answer**: Proxies intercept property access (get) and mutations (set). During the get trap, the system tracks which components/effects accessed which properties. During the set trap, it notifies only those tracked subscribers about changes. This creates a dependency graph automatically without manual subscriptions.

### Q2: What's the difference between Valtio and MobX?
**Answer**: Valtio uses immutable snapshots for reads with mutable writes, is lighter-weight, and React-focused. MobX uses decorators/annotations, has more features (reactions, computed caching), broader framework support, and requires wrapping components with observer(). Valtio is simpler; MobX is more powerful for complex scenarios.

### Q3: How does useSnapshot() maintain immutability?
**Answer**: useSnapshot() creates an immutable snapshot of the proxy state at render time. It uses structural sharing - unchanged parts reference the same objects while changed parts create new objects. This gives immutability benefits for React's reconciliation while the underlying proxy remains mutable.

### Q4: What are the performance implications of proxy-based state?
**Answer**: Proxies add overhead for every property access but enable fine-grained reactivity. This means fewer unnecessary re-renders compared to context or coarse-grained state. The overhead is negligible for most use cases but can matter for hot paths accessing properties millions of times per second.

### Q5: How do you handle async operations with proxy state?
**Answer**: Update the proxy state directly in async callbacks. For optimistic updates, mutate immediately then rollback on error. Use ref() for AbortControllers or other non-reactive objects. Consider separate loading/error state properties and batch updates when receiving data.

## Key Takeaways

1. Proxies enable automatic reactivity by intercepting property access and mutations
2. Valtio provides mutable writes with immutable snapshots for React
3. MobX offers powerful computed values and reactions with broader framework support
4. Fine-grained reactivity means components only re-render when their accessed properties change
5. Use ref() to exclude objects from reactivity (Maps, Sets, configs)
6. Proxy-based state feels like working with plain JavaScript objects
7. Snapshots maintain immutability for React's reconciliation
8. Computed/derived values automatically track dependencies
9. Performance is excellent for most use cases due to minimal re-renders
10. TypeScript support is excellent with proper type definitions

## Resources

- [Valtio Documentation](https://valtio.pmnd.rs/)
- [MobX Documentation](https://mobx.js.org/)
- [JavaScript Proxy MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [Valtio vs Zustand Comparison](https://github.com/pmndrs/valtio/wiki/How-does-Valtio-compare-to-Zustand)
- [MobX React Integration](https://mobx.js.org/react-integration.html)
- [Proxy Performance Characteristics](https://mathiasbynens.be/notes/es6-proxy-performance)
