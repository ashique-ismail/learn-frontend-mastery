# Elf - Reactive Immutable State Management

## The Idea

**In plain English:** Elf is a lightweight library that gives your Angular app a single, organized place to store and track all of its data (like a user's login status or a shopping cart), so every part of the app can read that data and automatically update whenever it changes.

**Real-world analogy:** Think of a school's main office whiteboard where teachers post class schedules and updates. Every classroom (component) can send a helper to check the whiteboard and immediately knows when something changes — instead of each classroom keeping its own messy notepad.

- The whiteboard = the Elf store (the one source of truth for app data)
- The class schedules written on it = the state (the actual data being tracked)
- A classroom sending a helper to watch for updates = a component subscribing to a store query

---

## Table of Contents

- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
- [Stores](#stores)
- [Queries](#queries)
- [Entities](#entities)
- [Persistence](#persistence)
- [DevTools](#devtools)
- [Advanced Features](#advanced-features)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Elf is a reactive immutable state management solution built on RxJS. It's modular, lightweight, and provides a functional API for managing state in Angular applications. Created by the author of Akita as a modern, tree-shakeable alternative.

**Key Features:**
- Modular architecture
- Tree-shakeable
- Functional API
- Entity management
- DevTools integration
- TypeScript-first
- Framework agnostic

## Installation & Setup

```bash
# Core package
npm install @ngneat/elf

# Entities support
npm install @ngneat/elf-entities

# Persistence
npm install @ngneat/elf-persist-state

# DevTools
npm install @ngneat/elf-devtools

# CLI
npm install @ngneat/elf-cli-ng --save-dev
```

### Basic Setup

```typescript
// todos.repository.ts
import { createStore, withProps, select } from '@ngneat/elf';
import { Injectable } from '@angular/core';

interface TodosProps {
  todos: Todo[];
  filter: string;
}

const todosStore = createStore(
  { name: 'todos' },
  withProps<TodosProps>({ todos: [], filter: '' })
);

@Injectable({ providedIn: 'root' })
export class TodosRepository {
  todos$ = todosStore.pipe(select((state) => state.todos));
  filter$ = todosStore.pipe(select((state) => state.filter));

  updateTodos(todos: Todo[]) {
    todosStore.update((state) => ({ ...state, todos }));
  }

  updateFilter(filter: string) {
    todosStore.update((state) => ({ ...state, filter }));
  }
}
```

## Core Concepts

### 1. Creating Stores

```typescript
import { createStore, withProps } from '@ngneat/elf';

// Simple store
interface CounterProps {
  count: number;
}

const counterStore = createStore(
  { name: 'counter' },
  withProps<CounterProps>({ count: 0 })
);

// Store with multiple properties
interface UserProps {
  user: User | null;
  loading: boolean;
  error: string | null;
}

const userStore = createStore(
  { name: 'user' },
  withProps<UserProps>({
    user: null,
    loading: false,
    error: null,
  })
);
```

### 2. Updating State

```typescript
import { createStore, withProps } from '@ngneat/elf';

const store = createStore(
  { name: 'counter' },
  withProps<{ count: number }>({ count: 0 })
);

// Update with new state
store.update((state) => ({
  ...state,
  count: state.count + 1,
}));

// Patch update (shallow merge)
import { setProp } from '@ngneat/elf';

store.update(setProp('count', 10));

// Multiple updates
import { setProps } from '@ngneat/elf';

store.update(
  setProps({
    count: 5,
    loading: false,
  })
);
```

### 3. Querying State

```typescript
import { select } from '@ngneat/elf';

// Select property
const count$ = store.pipe(select((state) => state.count));

// Select with computation
const doubled$ = store.pipe(
  select((state) => state.count * 2)
);

// Combine selections
import { combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';

const combined$ = combineLatest([
  store.pipe(select((state) => state.count)),
  store.pipe(select((state) => state.loading)),
]).pipe(
  map(([count, loading]) => ({ count, loading }))
);

// Get current value
const currentCount = store.getValue().count;

// Get entire state
const state = store.getValue();
```

## Stores

### 1. Store Configuration

```typescript
import { createStore, withProps } from '@ngneat/elf';

const store = createStore(
  {
    name: 'products',
    idKey: 'productId', // Custom ID key
  },
  withProps<ProductsProps>({ products: [] })
);
```

### 2. Multiple Store Features

```typescript
import {
  createStore,
  withProps,
  withEntities,
  withActiveId,
  withRequestsStatus,
} from '@ngneat/elf';

const store = createStore(
  { name: 'products' },
  withEntities<Product>(),
  withActiveId(),
  withProps<{ filter: string }>({ filter: '' }),
  withRequestsStatus()
);
```

### 3. Store Reset

```typescript
import { createStore, withProps } from '@ngneat/elf';

const store = createStore(
  { name: 'cart' },
  withProps<CartProps>({ items: [] })
);

// Reset to initial state
store.reset();

// Custom reset
store.update(() => ({
  items: [],
  total: 0,
}));
```

## Queries

### 1. Basic Queries

```typescript
import { createStore, withProps, select } from '@ngneat/elf';

const store = createStore(
  { name: 'user' },
  withProps<UserProps>({
    user: null,
    loading: false,
  })
);

@Injectable({ providedIn: 'root' })
export class UserRepository {
  // Select properties
  user$ = store.pipe(select((state) => state.user));
  loading$ = store.pipe(select((state) => state.loading));

  // Computed
  isLoggedIn$ = store.pipe(
    select((state) => state.user !== null)
  );

  userName$ = store.pipe(
    select((state) => state.user?.name ?? 'Guest')
  );
}
```

### 2. Advanced Selections

```typescript
import { select, distinctUntilArrayItemChanged } from '@ngneat/elf';
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ProductsRepository {
  // Debounced selection
  searchTerm$ = store.pipe(
    select((state) => state.searchTerm),
    debounceTime(300),
    distinctUntilChanged()
  );

  // Array change detection
  products$ = store.pipe(
    select((state) => state.products),
    distinctUntilArrayItemChanged()
  );

  // Complex computation
  stats$ = store.pipe(
    select((state) => ({
      total: state.products.length,
      inStock: state.products.filter((p) => p.inStock).length,
      avgPrice: state.products.reduce((sum, p) => sum + p.price, 0) / state.products.length,
    }))
  );
}
```

### 3. Combining Stores

```typescript
import { combineQueries } from '@ngneat/elf';

@Injectable({ providedIn: 'root' })
export class DashboardRepository {
  constructor(
    private userRepo: UserRepository,
    private orderRepo: OrderRepository
  ) {}

  dashboard$ = combineQueries([
    this.userRepo.user$,
    this.orderRepo.orders$,
  ]).pipe(
    map(([user, orders]) => ({
      userName: user?.name,
      orderCount: orders.length,
      recentOrders: orders.slice(0, 5),
    }))
  );
}
```

## Entities

### 1. Entity Store

```typescript
import {
  createStore,
  withEntities,
  selectAllEntities,
  setEntities,
  addEntities,
  updateEntities,
  deleteEntities,
} from '@ngneat/elf';
import { withEntitiesState } from '@ngneat/elf-entities';

interface Product {
  id: string;
  name: string;
  price: number;
}

const productsStore = createStore(
  { name: 'products' },
  withEntities<Product>()
);

@Injectable({ providedIn: 'root' })
export class ProductsRepository {
  products$ = productsStore.pipe(selectAllEntities());

  // Set all entities (replace)
  setProducts(products: Product[]) {
    productsStore.update(setEntities(products));
  }

  // Add entities
  addProduct(product: Product) {
    productsStore.update(addEntities(product));
  }

  addProducts(products: Product[]) {
    productsStore.update(addEntities(products));
  }

  // Update entities
  updateProduct(id: string, product: Partial<Product>) {
    productsStore.update(updateEntities(id, product));
  }

  // Delete entities
  deleteProduct(id: string) {
    productsStore.update(deleteEntities(id));
  }

  deleteProducts(ids: string[]) {
    productsStore.update(deleteEntities(ids));
  }
}
```

### 2. Entity Queries

```typescript
import {
  selectEntity,
  selectMany,
  selectAllEntities,
  selectAllEntitiesApply,
  selectEntityByPredicate,
  getEntity,
  getAllEntities,
  hasEntity,
} from '@ngneat/elf-entities';

@Injectable({ providedIn: 'root' })
export class ProductsRepository {
  // Select all
  products$ = productsStore.pipe(selectAllEntities());

  // Select by ID
  selectProduct(id: string) {
    return productsStore.pipe(selectEntity(id));
  }

  // Select many
  selectProducts(ids: string[]) {
    return productsStore.pipe(selectMany(ids));
  }

  // Select with filter
  activeProducts$ = productsStore.pipe(
    selectAllEntitiesApply({
      filterEntity: (product) => product.active,
    })
  );

  // Select with sort
  sortedProducts$ = productsStore.pipe(
    selectAllEntitiesApply({
      sortBy: 'price',
      sortByOrder: 'desc',
    })
  );

  // Select with limit
  topProducts$ = productsStore.pipe(
    selectAllEntitiesApply({
      filterEntity: (p) => p.active,
      sortBy: 'price',
      sortByOrder: 'desc',
      limitTo: 10,
    })
  );

  // Imperative queries
  getProduct(id: string): Product | undefined {
    return productsStore.query(getEntity(id));
  }

  getAllProducts(): Product[] {
    return productsStore.query(getAllEntities());
  }

  hasProduct(id: string): boolean {
    return productsStore.query(hasEntity(id));
  }
}
```

### 3. Active Entity

```typescript
import {
  createStore,
  withEntities,
  withActiveId,
  selectActiveEntity,
  setActiveId,
  resetActiveId,
  getActiveEntity,
} from '@ngneat/elf-entities';

const store = createStore(
  { name: 'products' },
  withEntities<Product>(),
  withActiveId()
);

@Injectable({ providedIn: 'root' })
export class ProductsRepository {
  activeProduct$ = store.pipe(selectActiveEntity());

  setActive(id: string) {
    store.update(setActiveId(id));
  }

  clearActive() {
    store.update(resetActiveId());
  }

  getActiveProduct(): Product | undefined {
    return store.query(getActiveEntity());
  }
}
```

## Persistence

### 1. Local Storage

```typescript
import { persistState, localStorageStrategy } from '@ngneat/elf-persist-state';

const store = createStore(
  { name: 'auth' },
  withProps<AuthProps>({ token: null })
);

// Persist entire store
export const persist = persistState(store, {
  key: 'auth',
  storage: localStorageStrategy,
});

// Persist specific properties
export const persist = persistState(store, {
  key: 'settings',
  storage: localStorageStrategy,
  source: () => store.pipe(select((state) => state.preferences)),
});
```

### 2. Session Storage

```typescript
import { sessionStorageStrategy } from '@ngneat/elf-persist-state';

export const persist = persistState(store, {
  key: 'cart',
  storage: sessionStorageStrategy,
});
```

### 3. Custom Storage

```typescript
import { createStorageStrategy } from '@ngneat/elf-persist-state';

const customStorage = createStorageStrategy({
  getItem(key: string) {
    return customStorageEngine.get(key);
  },
  setItem(key: string, value: string) {
    customStorageEngine.set(key, value);
  },
  removeItem(key: string) {
    customStorageEngine.remove(key);
  },
});

export const persist = persistState(store, {
  key: 'data',
  storage: customStorage,
});
```

### 4. Async Persistence

```typescript
import { persistState } from '@ngneat/elf-persist-state';

export const persist = persistState(store, {
  key: 'user',
  storage: localStorageStrategy,
  runGuard: () => inject(AuthService).isLoggedIn$,
  preStoreInit: (storageValue) => {
    // Migrate old data
    return {
      ...storageValue,
      version: 2,
    };
  },
});
```

## DevTools

### 1. Setup DevTools

```typescript
import { devTools } from '@ngneat/elf-devtools';

// Enable for all stores
if (!environment.production) {
  devTools();
}

// Configure
devTools({
  name: 'My App',
  maxAge: 25,
  logTrace: true,
});
```

### 2. Store Actions

```typescript
import { createStore, withProps } from '@ngneat/elf';

const store = createStore(
  { name: 'counter' },
  withProps<{ count: number }>({ count: 0 })
);

// Named updates for DevTools
store.update(
  (state) => ({ ...state, count: state.count + 1 }),
  { name: 'Increment Counter' } // Shows in DevTools
);

store.update(
  setProp('count', 0),
  { name: 'Reset Counter' }
);
```

## Advanced Features

### 1. Request Status

```typescript
import {
  createStore,
  withProps,
  withRequestsStatus,
  updateRequestStatus,
  selectRequestStatus,
  createRequestsStatusOperator,
} from '@ngneat/elf-requests';

const store = createStore(
  { name: 'users' },
  withProps<{ users: User[] }>({ users: [] }),
  withRequestsStatus()
);

@Injectable({ providedIn: 'root' })
export class UsersRepository {
  loading$ = store.pipe(selectRequestStatus('users'));

  loadUsers() {
    store.update(updateRequestStatus('users', 'pending'));

    return this.http.get<User[]>('/api/users').pipe(
      tap((users) => {
        store.update(
          setProps({ users }),
          updateRequestStatus('users', 'success')
        );
      }),
      catchError((error) => {
        store.update(updateRequestStatus('users', 'error'));
        return throwError(error);
      })
    );
  }

  // Or use operator
  loadUsersWithOperator() {
    return this.http.get<User[]>('/api/users').pipe(
      createRequestsStatusOperator(store, 'users'),
      tap((users) => store.update(setProps({ users })))
    );
  }
}
```

### 2. Pagination

```typescript
import {
  createStore,
  withEntities,
  withPagination,
  selectPaginationData,
  setCurrentPage,
  setPerPage,
} from '@ngneat/elf-pagination';

const store = createStore(
  { name: 'products' },
  withEntities<Product>(),
  withPagination()
);

@Injectable({ providedIn: 'root' })
export class ProductsRepository {
  pagination$ = store.pipe(selectPaginationData());

  setPage(page: number) {
    store.update(setCurrentPage(page));
  }

  setPageSize(perPage: number) {
    store.update(setPerPage(perPage));
  }

  loadPage(page: number) {
    this.setPage(page);
    
    const { perPage } = store.getValue();
    
    return this.http.get<Product[]>(`/api/products?page=${page}&perPage=${perPage}`).pipe(
      tap((products) => {
        store.update(setEntities(products));
      })
    );
  }
}
```

### 3. State History

```typescript
import { stateHistory } from '@ngneat/elf-state-history';

const store = createStore(
  { name: 'todos' },
  withProps<{ todos: Todo[] }>({ todos: [] })
);

const history = stateHistory(store, {
  maxAge: 10,
});

// Undo/Redo
history.undo();
history.redo();

// Check availability
const canUndo = history.hasPast;
const canRedo = history.hasFuture;

// Clear history
history.clear();

// Pause tracking
history.pause();
history.resume();

// Observables
history.hasPast$.subscribe(console.log);
history.hasFuture$.subscribe(console.log);
```

### 4. Batching Updates

```typescript
import { createStore, withProps } from '@ngneat/elf';

const store = createStore(
  { name: 'app' },
  withProps<AppProps>({ user: null, settings: null })
);

// Manual batching
store.update((state) => ({
  ...state,
  user: newUser,
  settings: newSettings,
  timestamp: Date.now(),
}));

// Multiple operations
import { setProps } from '@ngneat/elf';

store.update(
  setProps({
    user: newUser,
    settings: newSettings,
  })
);
```

## Performance Optimization

### 1. Memoization

```typescript
import { select } from '@ngneat/elf';
import { distinctUntilChanged } from 'rxjs/operators';

// Automatic memoization with distinctUntilChanged
const products$ = store.pipe(
  select((state) => state.products),
  distinctUntilChanged()
);

// Custom equality
import { distinctUntilArrayItemChanged } from '@ngneat/elf';

const products$ = store.pipe(
  select((state) => state.products),
  distinctUntilArrayItemChanged()
);
```

### 2. OnPush Strategy

```typescript
@Component({
  selector: 'app-products',
  template: `
    <div *ngFor="let product of products$ | async">
      {{ product.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductsComponent {
  products$ = this.productsRepo.products$;

  constructor(private productsRepo: ProductsRepository) {}
}
```

### 3. Selective Updates

```typescript
import { setProp } from '@ngneat/elf';

// Only update what changed
store.update(setProp('loading', true));

// Instead of
store.update((state) => ({ ...state, loading: true }));
```

## Common Mistakes

### 1. Mutating State

```typescript
// ❌ Wrong
store.update((state) => {
  state.items.push(newItem);
  return state;
});

// ✅ Correct
store.update((state) => ({
  ...state,
  items: [...state.items, newItem],
}));
```

### 2. Not Using Operators

```typescript
// ❌ Wrong
store.update((state) => ({
  ...state,
  count: 10,
}));

// ✅ Correct
import { setProp } from '@ngneat/elf';

store.update(setProp('count', 10));
```

### 3. Manual Subscriptions

```typescript
// ❌ Wrong
ngOnInit() {
  store.pipe(select((s) => s.data)).subscribe((data) => {
    this.data = data; // Memory leak
  });
}

// ✅ Correct
data$ = store.pipe(select((s) => s.data));
// Use async pipe
```

### 4. Forgetting DevTools Name

```typescript
// ❌ Wrong - unclear in DevTools
store.update((state) => ({ ...state, count: 1 }));

// ✅ Correct - clear action name
store.update(
  (state) => ({ ...state, count: 1 }),
  { name: 'Increment' }
);
```

### 5. Not Using Entity Features

```typescript
// ❌ Wrong - manual array management
const store = createStore(
  { name: 'products' },
  withProps<{ products: Product[] }>({ products: [] })
);

// ✅ Correct - use entities
const store = createStore(
  { name: 'products' },
  withEntities<Product>()
);
```

## Best Practices

### 1. Repository Pattern

```typescript
@Injectable({ providedIn: 'root' })
export class ProductsRepository {
  private store = createStore(
    { name: 'products' },
    withEntities<Product>()
  );

  products$ = this.store.pipe(selectAllEntities());

  setProducts(products: Product[]) {
    this.store.update(setEntities(products));
  }

  addProduct(product: Product) {
    this.store.update(addEntities(product));
  }
}
```

### 2. Named Updates

```typescript
// Always provide action names for DevTools
store.update(
  setProps({ loading: true }),
  { name: 'Start Loading' }
);

store.update(
  addEntities(products),
  { name: 'Load Products Success' }
);
```

### 3. Use Operators

```typescript
import { setProp, setProps } from '@ngneat/elf';
import { setEntities, addEntities, updateEntities } from '@ngneat/elf-entities';

// Leverage built-in operators
store.update(setProp('loading', true));
store.update(setEntities(products));
store.update(addEntities(product));
```

### 4. Type Safety

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
}

const store = createStore(
  { name: 'products' },
  withEntities<Product>() // Fully typed
);
```

### 5. Modular Stores

```typescript
// Combine features
const store = createStore(
  { name: 'products' },
  withEntities<Product>(),
  withActiveId(),
  withRequestsStatus(),
  withProps<{ filter: string }>({ filter: '' })
);
```

## When to Use

### Use Elf When

1. **Tree-shakeable bundle** - Want minimal bundle size
2. **Modular approach** - Only use features you need
3. **Functional API** - Prefer functional programming
4. **Modern codebase** - Starting fresh project
5. **Framework agnostic** - Need to use outside Angular

### Alternatives

- **Akita** - Similar but older, more plugins
- **NgRx Store** - More structure, larger ecosystem
- **NgRx Component Store** - Local state only
- **NGXS** - Class-based approach

## Interview Questions

### 1. What is Elf and how does it differ from Akita?

**Answer:** Elf is the successor to Akita, created by the same author. Key differences:
- **Tree-shakeable**: Only bundle what you use
- **Functional API**: Pure functions instead of classes
- **Modular**: Features are composable
- **Smaller bundle**: More lightweight
- **Modern**: Built with latest patterns

Both use RxJS, but Elf is more modern and efficient.

### 2. Explain the store creation pattern in Elf.

**Answer:**
```typescript
const store = createStore(
  { name: 'todos' }, // Config
  withProps<TodosProps>({ todos: [] }), // State
  withEntities<Todo>(), // Entity management
  withActiveId() // Active entity tracking
);
```
Stores are created by composing features (withProps, withEntities, etc.). This modular approach allows selecting only needed features.

### 3. How do you persist state in Elf?

**Answer:**
```typescript
import { persistState, localStorageStrategy } from '@ngneat/elf-persist-state';

const persist = persistState(store, {
  key: 'auth',
  storage: localStorageStrategy,
});
```
Elf provides persistence via plugins supporting localStorage, sessionStorage, or custom storage engines.

### 4. What are the benefits of Elf's operator functions?

**Answer:** Operator functions (setProp, setProps, setEntities, etc.) provide:
- **Immutability**: Guaranteed immutable updates
- **Type safety**: Full TypeScript inference
- **Readability**: Clear intent
- **Performance**: Optimized updates
- **DevTools**: Better action naming

### 5. How do you manage entities in Elf?

**Answer:**
```typescript
const store = createStore(
  { name: 'products' },
  withEntities<Product>()
);

// CRUD operations
store.update(setEntities(products)); // Set all
store.update(addEntities(product)); // Add
store.update(updateEntities(id, changes)); // Update
store.update(deleteEntities(id)); // Delete

// Queries
store.pipe(selectAllEntities());
store.pipe(selectEntity(id));
```
Built-in entity management with CRUD operations and queries.

## Key Takeaways

1. **Elf is modular and tree-shakeable** - only bundle features you use
2. **Functional API with operators** - setProp, setEntities, etc.
3. **Composable store features** - withProps, withEntities, withActiveId
4. **Built-in entity management** - CRUD operations and queries
5. **Persistence support** - localStorage, sessionStorage, custom storage
6. **DevTools integration** - time-travel debugging with named actions
7. **TypeScript-first** - full type inference and safety
8. **Framework agnostic** - works outside Angular too
9. **Lightweight** - smaller bundle than alternatives
10. **Modern patterns** - built with latest best practices

## Resources

### Official Documentation
- [Elf Docs](https://ngneat.github.io/elf/)
- [API Reference](https://ngneat.github.io/elf/docs/api/core/)
- [Examples](https://github.com/ngneat/elf/tree/master/apps/ng-app)

### Tutorials
- [Introduction to Elf](https://blog.nrwl.io/introducing-elf-a-reactive-immutable-store-for-javascript-applications-f1f87a3f6a5a)
- [Migration from Akita](https://ngneat.github.io/elf/docs/migration/)

### Tools
- [Elf CLI](https://ngneat.github.io/elf/docs/cli/)
- [DevTools](https://ngneat.github.io/elf/docs/dev-tools/)

### Community
- [GitHub Repository](https://github.com/ngneat/elf)
- [Discussions](https://github.com/ngneat/elf/discussions)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/ngneat-elf)
