# Akita - State Management for Angular

## The Idea

**In plain English:** Akita is a tool for Angular apps that acts like a shared memory bank — it stores all the important data your app needs in one organized place, and any part of the app can read from or write to it without things getting messy. "State" just means the current snapshot of your app's data, like which user is logged in or what items are in a cart.

**Real-world analogy:** Think of a busy restaurant with a central whiteboard in the kitchen. Waiters write orders on it, the chef reads from it to cook, and the cashier reads from it to charge customers — everyone works from the same source of truth without shouting across the room.

- The whiteboard = the Store (holds all current data)
- The act of reading the whiteboard = the Query (retrieves specific data for whoever needs it)
- The waiter writing an order = the Service (updates the store when something changes)

---

## Table of Contents

- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
- [Stores](#stores)
- [Queries](#queries)
- [Entities](#entities)
- [Transactions](#transactions)
- [Plugins](#plugins)
- [Advanced Features](#advanced-features)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Akita is a state management pattern built on top of RxJS that provides a simple and powerful way to manage state in Angular applications. It's designed to be simple, boilerplate-free, and feature-rich.

**Key Features:**
- Simple and intuitive API
- Entity management built-in
- Plugins ecosystem
- DevTools integration
- TypeScript support
- Transactions and batching

## Installation & Setup

```bash
# Install Akita
npm install @datorama/akita

# Install DevTools
npm install @datorama/akita-ng-devtools --save-dev

# CLI for code generation
npm install @datorama/akita-cli --save-dev
```

### Basic Setup

```typescript
// app.module.ts
import { AkitaNgDevtools } from '@datorama/akita-ng-devtools';
import { environment } from '../environments/environment';

@NgModule({
  imports: [
    environment.production ? [] : AkitaNgDevtools.forRoot(),
  ],
})
export class AppModule {}
```

### Create Store and Query

```typescript
// session.store.ts
import { Store, StoreConfig } from '@datorama/akita';

export interface SessionState {
  token: string;
  username: string;
}

export function createInitialState(): SessionState {
  return {
    token: '',
    username: '',
  };
}

@Injectable({ providedIn: 'root' })
@StoreConfig({ name: 'session' })
export class SessionStore extends Store<SessionState> {
  constructor() {
    super(createInitialState());
  }
}

// session.query.ts
import { Query } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class SessionQuery extends Query<SessionState> {
  isLoggedIn$ = this.select((state) => !!state.token);
  username$ = this.select((state) => state.username);

  constructor(protected store: SessionStore) {
    super(store);
  }
}

// session.service.ts
@Injectable({ providedIn: 'root' })
export class SessionService {
  constructor(private sessionStore: SessionStore) {}

  login(token: string, username: string) {
    this.sessionStore.update({ token, username });
  }

  logout() {
    this.sessionStore.update(createInitialState());
  }
}
```

## Core Concepts

### 1. Store

Stores hold application state:

```typescript
// cart.store.ts
import { Store, StoreConfig } from '@datorama/akita';

export interface CartState {
  items: CartItem[];
  total: number;
  loading: boolean;
}

const initialState: CartState = {
  items: [],
  total: 0,
  loading: false,
};

@Injectable({ providedIn: 'root' })
@StoreConfig({ name: 'cart' })
export class CartStore extends Store<CartState> {
  constructor() {
    super(initialState);
  }
}
```

### 2. Query

Queries select and combine state:

```typescript
// cart.query.ts
import { Query } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class CartQuery extends Query<CartState> {
  // Select entire state
  cart$ = this.select();

  // Select specific properties
  items$ = this.select('items');
  total$ = this.select('total');
  loading$ = this.select('loading');

  // Computed properties
  itemCount$ = this.select((state) => state.items.length);
  isEmpty$ = this.select((state) => state.items.length === 0);

  // Combine selectors
  cartSummary$ = this.select(['items', 'total'], (items, total) => ({
    items,
    total,
    count: items.length,
  }));

  constructor(protected store: CartStore) {
    super(store);
  }

  // Imperative queries
  getTotal(): number {
    return this.getValue().total;
  }

  hasItems(): boolean {
    return this.getValue().items.length > 0;
  }
}
```

### 3. Service

Services contain business logic:

```typescript
// cart.service.ts
@Injectable({ providedIn: 'root' })
export class CartService {
  constructor(
    private cartStore: CartStore,
    private cartQuery: CartQuery
  ) {}

  addItem(item: CartItem) {
    const currentItems = this.cartQuery.getValue().items;
    this.cartStore.update((state) => ({
      items: [...currentItems, item],
      total: state.total + item.price,
    }));
  }

  removeItem(id: string) {
    const items = this.cartQuery.getValue().items;
    const item = items.find((i) => i.id === id);
    
    if (item) {
      this.cartStore.update((state) => ({
        items: items.filter((i) => i.id !== id),
        total: state.total - item.price,
      }));
    }
  }

  clear() {
    this.cartStore.update({
      items: [],
      total: 0,
    });
  }

  setLoading(loading: boolean) {
    this.cartStore.setLoading(loading);
  }
}
```

## Stores

### 1. Updating Store

```typescript
@Injectable({ providedIn: 'root' })
export class TodoService {
  constructor(private todoStore: TodoStore) {}

  // Simple update
  updateTitle(title: string) {
    this.todoStore.update({ title });
  }

  // Function update
  incrementCount() {
    this.todoStore.update((state) => ({
      count: state.count + 1,
    }));
  }

  // Partial update
  updatePartial(partial: Partial<TodoState>) {
    this.todoStore.update(partial);
  }

  // Reset to initial
  reset() {
    this.todoStore.reset();
  }
}
```

### 2. Store Config

```typescript
@StoreConfig({
  name: 'todos',
  resettable: true,
  cache: {
    ttl: 3600000, // 1 hour
  },
})
export class TodoStore extends Store<TodoState> {}
```

### 3. Active State

Track active entity:

```typescript
@StoreConfig({ name: 'products' })
export class ProductStore extends Store<ProductState> {
  constructor() {
    super(initialState);
  }
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(
    private productStore: ProductStore,
    private productQuery: ProductQuery
  ) {}

  setActive(id: string) {
    this.productStore.setActive(id);
  }

  clearActive() {
    this.productStore.setActive(null);
  }
}

// In Query
export class ProductQuery extends Query<ProductState> {
  activeProduct$ = this.selectActive();
  
  constructor(protected store: ProductStore) {
    super(store);
  }
}
```

## Queries

### 1. Select Methods

```typescript
@Injectable({ providedIn: 'root' })
export class UserQuery extends Query<UserState> {
  constructor(protected store: UserStore) {
    super(store);
  }

  // Select property
  username$ = this.select('username');

  // Select with function
  isAdmin$ = this.select((state) => state.role === 'admin');

  // Select multiple
  userInfo$ = this.select(['username', 'email']);

  // Select with custom operator
  debouncedSearch$ = this.select('searchTerm').pipe(
    debounceTime(300),
    distinctUntilChanged()
  );

  // waitForActive - wait until entity is active
  activeUser$ = this.selectActive().pipe(
    filterNil() // Akita operator
  );
}
```

### 2. Combining Queries

```typescript
@Injectable({ providedIn: 'root' })
export class DashboardQuery {
  constructor(
    private userQuery: UserQuery,
    private orderQuery: OrderQuery,
    private productQuery: ProductQuery
  ) {}

  dashboard$ = combineQueries([
    this.userQuery.select(),
    this.orderQuery.selectAll(),
    this.productQuery.selectAll(),
  ]).pipe(
    map(([user, orders, products]) => ({
      user,
      totalOrders: orders.length,
      totalProducts: products.length,
      recentOrders: orders.slice(0, 5),
    }))
  );
}
```

### 3. Query Filters

```typescript
@Injectable({ providedIn: 'root' })
export class ProductQuery extends QueryEntity<ProductState> {
  constructor(protected store: ProductStore) {
    super(store);
  }

  // Filter active products
  activeProducts$ = this.selectAll({
    filterBy: (product) => product.active,
  });

  // Sort products
  sortedProducts$ = this.selectAll({
    sortBy: 'name',
    sortByOrder: Order.ASC,
  });

  // Limit results
  topProducts$ = this.selectAll({
    limitTo: 10,
  });

  // Combine filters
  filteredProducts$ = this.selectAll({
    filterBy: (product) => product.active && product.inStock,
    sortBy: 'price',
    sortByOrder: Order.DESC,
    limitTo: 20,
  });
}
```

## Entities

### 1. Entity Store

```typescript
// products.store.ts
import { EntityStore, EntityState, StoreConfig } from '@datorama/akita';

export interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

export interface ProductsState extends EntityState<Product, string> {
  filter: string;
  loading: boolean;
}

@Injectable({ providedIn: 'root' })
@StoreConfig({ name: 'products' })
export class ProductsStore extends EntityStore<ProductsState> {
  constructor() {
    super({
      filter: '',
      loading: false,
    });
  }
}
```

### 2. Entity Query

```typescript
// products.query.ts
import { QueryEntity } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class ProductsQuery extends QueryEntity<ProductsState> {
  // Select all entities
  allProducts$ = this.selectAll();

  // Select by ID
  selectProduct(id: string) {
    return this.selectEntity(id);
  }

  // Select many
  selectProducts(ids: string[]) {
    return this.selectMany(ids);
  }

  // Select active
  activeProduct$ = this.selectActive();

  // Count
  productCount$ = this.selectCount();

  // Has entity
  hasProduct(id: string) {
    return this.hasEntity(id);
  }

  constructor(protected store: ProductsStore) {
    super(store);
  }
}
```

### 3. Entity CRUD Operations

```typescript
// products.service.ts
@Injectable({ providedIn: 'root' })
export class ProductsService {
  constructor(
    private productsStore: ProductsStore,
    private http: HttpClient
  ) {}

  // Add entity
  add(product: Product) {
    this.productsStore.add(product);
  }

  // Add many
  addMany(products: Product[]) {
    this.productsStore.add(products);
  }

  // Update entity
  update(id: string, product: Partial<Product>) {
    this.productsStore.update(id, product);
  }

  // Update many
  updateMany(ids: string[], product: Partial<Product>) {
    this.productsStore.update(ids, product);
  }

  // Upsert (add or update)
  upsert(id: string, product: Product) {
    this.productsStore.upsert(id, product);
  }

  // Remove entity
  remove(id: string) {
    this.productsStore.remove(id);
  }

  // Remove many
  removeMany(ids: string[]) {
    this.productsStore.remove(ids);
  }

  // Set entities (replace all)
  set(products: Product[]) {
    this.productsStore.set(products);
  }

  // Load from API
  load() {
    this.productsStore.setLoading(true);
    
    return this.http.get<Product[]>('/api/products').pipe(
      tap((products) => {
        this.productsStore.set(products);
        this.productsStore.setLoading(false);
      })
    );
  }
}
```

## Transactions

### 1. Basic Transactions

```typescript
import { applyTransaction } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class OrderService {
  constructor(
    private orderStore: OrderStore,
    private cartStore: CartStore,
    private userStore: UserStore
  ) {}

  checkout() {
    applyTransaction(() => {
      // All updates batched into single emission
      this.orderStore.add(newOrder);
      this.cartStore.update({ items: [] });
      this.userStore.update({ lastOrderDate: new Date() });
    });
  }
}
```

### 2. Conditional Transactions

```typescript
@Injectable({ providedIn: 'root' })
export class InventoryService {
  constructor(
    private productStore: ProductsStore,
    private orderStore: OrderStore
  ) {}

  purchaseProduct(productId: string, quantity: number) {
    const product = this.productQuery.getEntity(productId);
    
    if (!product || product.stock < quantity) {
      return throwError('Insufficient stock');
    }

    applyTransaction(() => {
      this.productStore.update(productId, {
        stock: product.stock - quantity,
      });
      
      this.orderStore.add({
        id: guid(),
        productId,
        quantity,
        date: new Date(),
      });
    });

    return of(true);
  }
}
```

## Plugins

### 1. Persist State

```typescript
import { persistState } from '@datorama/akita';

// Persist entire store
export const sessionPersist = persistState({
  key: 'session',
});

// Persist specific stores
export const multiPersist = persistState({
  include: ['session', 'user', 'settings'],
  key: 'app-state',
});

// Custom storage
export const customPersist = persistState({
  key: 'session',
  storage: customStorageEngine,
});
```

### 2. Entity State History (Undo/Redo)

```typescript
import { StateHistoryPlugin } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class TodoService {
  private history = new StateHistoryPlugin(this.todoQuery);

  constructor(
    private todoStore: TodoStore,
    private todoQuery: TodoQuery
  ) {}

  undo() {
    this.history.undo();
  }

  redo() {
    this.history.redo();
  }

  canUndo(): boolean {
    return this.history.hasPast;
  }

  canRedo(): boolean {
    return this.history.hasFuture;
  }

  clearHistory() {
    this.history.clear();
  }
}
```

### 3. Dirty Check

```typescript
import { DirtyCheckPlugin } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class FormService {
  private dirtyCheck = new DirtyCheckPlugin(this.formQuery);

  constructor(
    private formStore: FormStore,
    private formQuery: FormQuery
  ) {}

  setHead() {
    this.dirtyCheck.setHead();
  }

  isDirty(): boolean {
    return this.dirtyCheck.isDirty();
  }

  reset() {
    this.dirtyCheck.reset();
  }

  hasChanges$ = this.dirtyCheck.isDirty$;
}
```

### 4. Pagination

```typescript
import { PaginatorPlugin } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private paginator = new PaginatorPlugin(this.productsQuery).withControls();

  constructor(
    private productsStore: ProductsStore,
    private productsQuery: ProductsQuery,
    private http: HttpClient
  ) {}

  page$ = this.paginator.pageChanges.pipe(
    switchMap((page) => 
      this.http.get<Product[]>(`/api/products?page=${page}`)
    )
  );

  nextPage() {
    this.paginator.nextPage();
  }

  prevPage() {
    this.paginator.prevPage();
  }

  setPage(page: number) {
    this.paginator.setPage(page);
  }
}
```

## Advanced Features

### 1. Array Utilities

```typescript
import { arrayAdd, arrayRemove, arrayUpdate } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class TodoService {
  constructor(private todoStore: TodoStore) {}

  addTodo(todo: Todo) {
    this.todoStore.update((state) => ({
      todos: arrayAdd(state.todos, todo),
    }));
  }

  removeTodo(id: string) {
    this.todoStore.update((state) => ({
      todos: arrayRemove(state.todos, id),
    }));
  }

  updateTodo(id: string, todo: Partial<Todo>) {
    this.todoStore.update((state) => ({
      todos: arrayUpdate(state.todos, id, todo),
    }));
  }

  // Prepend
  prependTodo(todo: Todo) {
    this.todoStore.update((state) => ({
      todos: arrayAdd(state.todos, todo, { prepend: true }),
    }));
  }
}
```

### 2. Object Utilities

```typescript
import { setProps } from '@datorama/akita';

@Injectable({ providedIn: 'root' })
export class SettingsService {
  constructor(private settingsStore: SettingsStore) {}

  updateTheme(theme: string) {
    this.settingsStore.update(
      setProps({
        ui: {
          theme,
        },
      })
    );
  }

  updateNestedProperty(path: string, value: any) {
    this.settingsStore.update(
      setProps({
        [path]: value,
      })
    );
  }
}
```

### 3. Entity ID Strategies

```typescript
// UUID strategy
@StoreConfig({
  name: 'products',
  idKey: 'productId', // Custom ID key
})
export class ProductsStore extends EntityStore<ProductsState> {}

// Composite key
@StoreConfig({
  name: 'orders',
  idKey: ['userId', 'orderId'],
})
export class OrdersStore extends EntityStore<OrdersState> {}
```

### 4. Custom Store Methods

```typescript
@Injectable({ providedIn: 'root' })
@StoreConfig({ name: 'counter' })
export class CounterStore extends Store<CounterState> {
  constructor() {
    super({ count: 0 });
  }

  increment() {
    this.update((state) => ({ count: state.count + 1 }));
  }

  decrement() {
    this.update((state) => ({ count: state.count - 1 }));
  }

  incrementBy(amount: number) {
    this.update((state) => ({ count: state.count + amount }));
  }

  reset() {
    this.update({ count: 0 });
  }
}
```

## Performance Optimization

### 1. Memoization

```typescript
// Queries are automatically memoized
@Injectable({ providedIn: 'root' })
export class ProductsQuery extends QueryEntity<ProductsState> {
  // Only recomputes when products change
  expensiveComputation$ = this.selectAll().pipe(
    map((products) => {
      console.log('Computing...');
      return products.map((p) => ({
        ...p,
        computed: heavyComputation(p),
      }));
    })
  );

  constructor(protected store: ProductsStore) {
    super(store);
  }
}
```

### 2. OnPush Detection

```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products$ | async">
      {{ product.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductListComponent {
  products$ = this.productsQuery.selectAll();

  constructor(private productsQuery: ProductsQuery) {}
}
```

### 3. Limit Selections

```typescript
@Injectable({ providedIn: 'root' })
export class ProductsQuery extends QueryEntity<ProductsState> {
  // Only select top 10
  topProducts$ = this.selectAll({
    limitTo: 10,
    sortBy: 'price',
    sortByOrder: Order.DESC,
  });

  constructor(protected store: ProductsStore) {
    super(store);
  }
}
```

## Common Mistakes

### 1. Mutating State

```typescript
// ❌ Wrong
update() {
  const state = this.getValue();
  state.items.push(newItem); // Mutation!
  this.update(state);
}

// ✅ Correct
update() {
  this.update((state) => ({
    items: [...state.items, newItem],
  }));
}
```

### 2. Not Using Queries

```typescript
// ❌ Wrong
constructor(private store: ProductsStore) {
  this.products$ = this.store._value().entities;
}

// ✅ Correct
constructor(private query: ProductsQuery) {
  this.products$ = this.query.selectAll();
}
```

### 3. Manual Subscriptions

```typescript
// ❌ Wrong
ngOnInit() {
  this.query.selectAll().subscribe((products) => {
    this.products = products; // Memory leak
  });
}

// ✅ Correct
products$ = this.query.selectAll();
// Use async pipe in template
```

### 4. Forgetting Entity Store

```typescript
// ❌ Wrong - using regular Store for entities
@StoreConfig({ name: 'products' })
export class ProductsStore extends Store<{ products: Product[] }> {}

// ✅ Correct - use EntityStore
@StoreConfig({ name: 'products' })
export class ProductsStore extends EntityStore<ProductsState> {}
```

### 5. Not Using Transactions

```typescript
// ❌ Wrong - multiple emissions
updateMultiple() {
  this.store1.update({ value: 1 });
  this.store2.update({ value: 2 });
  this.store3.update({ value: 3 });
}

// ✅ Correct - single emission
updateMultiple() {
  applyTransaction(() => {
    this.store1.update({ value: 1 });
    this.store2.update({ value: 2 });
    this.store3.update({ value: 3 });
  });
}
```

## Best Practices

### 1. Organize by Feature

```
src/
  app/
    products/
      state/
        products.model.ts
        products.store.ts
        products.query.ts
        products.service.ts
```

### 2. Use Services for Logic

```typescript
// Service contains all business logic
@Injectable({ providedIn: 'root' })
export class ProductsService {
  constructor(
    private productsStore: ProductsStore,
    private productsQuery: ProductsQuery,
    private http: HttpClient
  ) {}

  load() {
    return this.http.get<Product[]>('/api/products').pipe(
      tap((products) => this.productsStore.set(products))
    );
  }
}
```

### 3. Type Everything

```typescript
export interface Product {
  id: string;
  name: string;
  price: number;
}

export interface ProductsState extends EntityState<Product, string> {
  ui: {
    filter: string;
    sortBy: string;
  };
}
```

### 4. Use Plugins

```typescript
// Leverage built-in plugins
const persistConfig = persistState({
  include: ['session', 'settings'],
});

const history = new StateHistoryPlugin(query);
const paginator = new PaginatorPlugin(query);
```

### 5. DevTools in Development

```typescript
// Enable DevTools in development only
imports: [
  environment.production ? [] : AkitaNgDevtools.forRoot(),
]
```

## When to Use

### Use Akita When

1. **Less boilerplate** - Want simpler API than NgRx
2. **Entity management** - Working with collections
3. **Plugin ecosystem** - Need persistence, pagination, etc.
4. **Quick setup** - Want to get started fast
5. **Medium-sized apps** - Don't need enterprise patterns

### Alternatives

- **NgRx Store** - More structure, larger apps
- **NgRx Component Store** - Local state only
- **NGXS** - Similar simplicity
- **Elf** - Newer, modular approach

## Interview Questions

### 1. What is Akita and how does it differ from NgRx?

**Answer:** Akita is a state management library built on RxJS with less boilerplate than NgRx:
- **No actions/reducers**: Direct store updates
- **Entity management**: Built-in EntityStore
- **Plugins**: Persistence, pagination, history
- **Simpler API**: Less code to write
- **Query pattern**: Separate queries from stores

NgRx has more structure and is better for very large apps, while Akita is simpler and faster to implement.

### 2. Explain the Store-Query pattern in Akita.

**Answer:**
- **Store**: Holds and updates state (write operations)
- **Query**: Selects and derives state (read operations)
- **Service**: Contains business logic, uses Store and Query

This separation provides clear responsibilities and makes testing easier.

### 3. What are transactions in Akita?

**Answer:** Transactions batch multiple store updates into a single emission:
```typescript
applyTransaction(() => {
  this.store1.update({ value: 1 });
  this.store2.update({ value: 2 });
});
```
Benefits: better performance, atomic updates, single notification to subscribers.

### 4. How do you manage collections in Akita?

**Answer:** Use EntityStore and QueryEntity:
```typescript
export class ProductsStore extends EntityStore<ProductsState> {}
export class ProductsQuery extends QueryEntity<ProductsState> {}
```
Provides built-in CRUD: add, update, remove, upsert, plus selectors: selectAll, selectEntity, selectMany.

### 5. What Akita plugins are commonly used?

**Answer:**
- **persistState**: LocalStorage/SessionStorage persistence
- **StateHistoryPlugin**: Undo/redo functionality
- **PaginatorPlugin**: Pagination support
- **DirtyCheckPlugin**: Track form changes
- **EntityCollectionPlugin**: Collection management

## Key Takeaways

1. **Akita uses Store-Query-Service pattern** for clear separation of concerns
2. **EntityStore simplifies collection management** with built-in CRUD operations
3. **Transactions batch updates** for better performance and atomicity
4. **Plugins extend functionality** - persistence, pagination, history, dirty checking
5. **Less boilerplate than NgRx** - no actions or reducers required
6. **Built on RxJS** - leverages observables for reactive programming
7. **TypeScript-first** - excellent type inference and safety
8. **DevTools integration** for debugging and time-travel
9. **Array/object utilities** simplify immutable updates
10. **Queries are memoized** - only recompute when dependencies change

## Resources

### Official Documentation
- [Akita Docs](https://datorama.github.io/akita/)
- [API Reference](https://datorama.github.io/akita/docs/api/store)
- [Examples](https://github.com/datorama/akita/tree/master/examples)

### Tutorials
- [Akita Introduction](https://netbasal.com/introducing-akita-a-new-state-management-pattern-for-angular-applications-f2f0fab5a8)
- [Entity Management](https://netbasal.com/manage-your-entities-with-akita-like-a-boss-768c2d8c9d1a)

### Tools
- [Akita CLI](https://github.com/datorama/akita-cli)
- [Angular CLI Schematics](https://github.com/datorama/akita-schematics)

### Community
- [GitHub Repository](https://github.com/datorama/akita)
- [Gitter Chat](https://gitter.im/akita-state-management/Lobby)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/akita)
