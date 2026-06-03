# NgRx Component Store - Local State Management for Angular

## Table of Contents
- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
- [State Management](#state-management)
- [Updaters](#updaters)
- [Selectors](#selectors)
- [Effects](#effects)
- [Advanced Features](#advanced-features)
- [Integration](#integration)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

NgRx Component Store is a standalone library for managing local component state in Angular. Unlike NgRx Store (global state), Component Store is designed for component-level or feature-level state management with a simpler API.

**Key Features:**
- Local state management
- Reactive programming with RxJS
- Built-in lifecycle management
- Effects for side effects
- Lazy initialization
- TypeScript support

## Installation & Setup

```bash
# Install Component Store
npm install @ngrx/component-store

# Usually paired with RxJS
npm install rxjs
```

### Basic Setup

```typescript
// movie.store.ts
import { Injectable } from '@angular/core';
import { ComponentStore } from '@ngrx/component-store';

export interface Movie {
  id: string;
  title: string;
  year: number;
}

export interface MovieState {
  movies: Movie[];
  loading: boolean;
  filter: string;
}

const initialState: MovieState = {
  movies: [],
  loading: false,
  filter: '',
};

@Injectable()
export class MovieStore extends ComponentStore<MovieState> {
  constructor() {
    super(initialState);
  }
}
```

### Using in Component

```typescript
// movie-list.component.ts
import { Component } from '@angular/core';
import { MovieStore } from './movie.store';

@Component({
  selector: 'app-movie-list',
  template: `
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngFor="let movie of movies$ | async">
      {{ movie.title }} ({{ movie.year }})
    </div>
  `,
  providers: [MovieStore], // Provide at component level
})
export class MovieListComponent {
  movies$ = this.movieStore.movies$;
  loading$ = this.movieStore.loading$;

  constructor(private movieStore: MovieStore) {}
}
```

## Core Concepts

### 1. State Interface

Define your state structure:

```typescript
// user.store.ts
export interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
}

export interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
  lastUpdated: Date | null;
}

const initialState: UserState = {
  user: null,
  loading: false,
  error: null,
  lastUpdated: null,
};

@Injectable()
export class UserStore extends ComponentStore<UserState> {
  constructor() {
    super(initialState);
  }
}
```

### 2. Reading State

Access state using selectors:

```typescript
@Injectable()
export class UserStore extends ComponentStore<UserState> {
  // Select entire state
  readonly state$ = this.select((state) => state);

  // Select specific properties
  readonly user$ = this.select((state) => state.user);
  readonly loading$ = this.select((state) => state.loading);
  readonly error$ = this.select((state) => state.error);

  // Composed selectors
  readonly isLoggedIn$ = this.select(
    this.user$,
    (user) => user !== null
  );

  readonly userName$ = this.select(
    this.user$,
    (user) => user?.name ?? 'Guest'
  );

  constructor() {
    super(initialState);
  }
}
```

### 3. Imperative State Access

Get state synchronously when needed:

```typescript
@Injectable()
export class UserStore extends ComponentStore<UserState> {
  constructor() {
    super(initialState);
  }

  getCurrentUser(): User | null {
    return this.get().user; // Synchronous access
  }

  isLoading(): boolean {
    return this.get().loading;
  }

  logCurrentState(): void {
    console.log('Current state:', this.get());
  }
}
```

## State Management

### 1. Updaters (State Mutations)

Update state with updaters:

```typescript
@Injectable()
export class UserStore extends ComponentStore<UserState> {
  constructor() {
    super(initialState);
  }

  // Simple updater
  readonly setLoading = this.updater((state, loading: boolean) => ({
    ...state,
    loading,
  }));

  // Updater with object
  readonly setUser = this.updater((state, user: User) => ({
    ...state,
    user,
    loading: false,
    error: null,
    lastUpdated: new Date(),
  }));

  // Updater with transformation
  readonly updateUserName = this.updater((state, name: string) => ({
    ...state,
    user: state.user ? { ...state.user, name } : null,
  }));

  // Clear user
  readonly clearUser = this.updater((state) => ({
    ...state,
    user: null,
  }));

  // Set error
  readonly setError = this.updater((state, error: string) => ({
    ...state,
    loading: false,
    error,
  }));
}

// Usage in component
this.userStore.setUser({ id: '1', name: 'John', email: 'john@example.com' });
this.userStore.setLoading(true);
this.userStore.updateUserName('Jane');
```

### 2. Updater Patterns

Common updater patterns:

```typescript
@Injectable()
export class TodoStore extends ComponentStore<TodoState> {
  constructor() {
    super(initialState);
  }

  // Add item to array
  readonly addTodo = this.updater((state, todo: Todo) => ({
    ...state,
    todos: [...state.todos, todo],
  }));

  // Update item in array
  readonly updateTodo = this.updater((state, updatedTodo: Todo) => ({
    ...state,
    todos: state.todos.map((todo) =>
      todo.id === updatedTodo.id ? updatedTodo : todo
    ),
  }));

  // Remove item from array
  readonly removeTodo = this.updater((state, id: string) => ({
    ...state,
    todos: state.todos.filter((todo) => todo.id !== id),
  }));

  // Toggle boolean
  readonly toggleComplete = this.updater((state, id: string) => ({
    ...state,
    todos: state.todos.map((todo) =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ),
  }));

  // Patch state
  readonly patchState = this.updater((state, partial: Partial<TodoState>) => ({
    ...state,
    ...partial,
  }));
}
```

### 3. Batch Updates

Use patchState for multiple updates:

```typescript
@Injectable()
export class ProductStore extends ComponentStore<ProductState> {
  constructor() {
    super(initialState);
  }

  // Multiple updates in one call
  readonly loadProductsSuccess = this.updater((state, products: Product[]) => ({
    ...state,
    products,
    loading: false,
    error: null,
    lastUpdated: new Date(),
  }));

  // Or use built-in patchState
  readonly updateMultipleFields = this.updater((state, updates: Partial<ProductState>) => {
    this.patchState(updates); // Helper method
    return state;
  });
}
```

## Updaters

### 1. Simple Updaters

```typescript
@Injectable()
export class CounterStore extends ComponentStore<{ count: number }> {
  constructor() {
    super({ count: 0 });
  }

  // Increment
  readonly increment = this.updater((state) => ({
    count: state.count + 1,
  }));

  // Decrement
  readonly decrement = this.updater((state) => ({
    count: state.count - 1,
  }));

  // Reset
  readonly reset = this.updater(() => ({ count: 0 }));

  // Set specific value
  readonly setCount = this.updater((state, count: number) => ({ count }));

  // Increment by amount
  readonly incrementBy = this.updater((state, amount: number) => ({
    count: state.count + amount,
  }));
}
```

### 2. Updaters with Observables

Updaters can accept observables:

```typescript
@Injectable()
export class SearchStore extends ComponentStore<SearchState> {
  constructor() {
    super(initialState);
  }

  // Updater accepting observable
  readonly updateQuery = this.updater((state, query: string) => ({
    ...state,
    query,
    loading: true,
  }));

  // Connect observable to updater
  connectQueryStream(query$: Observable<string>): void {
    this.updateQuery(query$);
  }
}

// In component
ngOnInit() {
  const searchInput$ = this.searchControl.valueChanges;
  this.searchStore.connectQueryStream(searchInput$);
}
```

### 3. Conditional Updates

```typescript
@Injectable()
export class CartStore extends ComponentStore<CartState> {
  constructor() {
    super(initialState);
  }

  readonly addToCart = this.updater((state, product: Product) => {
    // Check if product already in cart
    const existingItem = state.items.find((item) => item.id === product.id);

    if (existingItem) {
      // Update quantity
      return {
        ...state,
        items: state.items.map((item) =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        ),
      };
    } else {
      // Add new item
      return {
        ...state,
        items: [...state.items, { ...product, quantity: 1 }],
      };
    }
  });
}
```

## Selectors

### 1. Basic Selectors

```typescript
@Injectable()
export class ProductStore extends ComponentStore<ProductState> {
  constructor() {
    super(initialState);
  }

  // Simple selectors
  readonly products$ = this.select((state) => state.products);
  readonly loading$ = this.select((state) => state.loading);
  readonly filter$ = this.select((state) => state.filter);

  // Computed selectors
  readonly productCount$ = this.select(
    this.products$,
    (products) => products.length
  );

  readonly hasProducts$ = this.select(
    this.products$,
    (products) => products.length > 0
  );
}
```

### 2. Combining Selectors

```typescript
@Injectable()
export class ProductStore extends ComponentStore<ProductState> {
  constructor() {
    super(initialState);
  }

  readonly products$ = this.select((state) => state.products);
  readonly filter$ = this.select((state) => state.filter);
  readonly sortBy$ = this.select((state) => state.sortBy);

  // Combine multiple selectors
  readonly filteredProducts$ = this.select(
    this.products$,
    this.filter$,
    (products, filter) => {
      if (!filter) return products;
      return products.filter((p) =>
        p.name.toLowerCase().includes(filter.toLowerCase())
      );
    }
  );

  // Complex combination
  readonly sortedFilteredProducts$ = this.select(
    this.filteredProducts$,
    this.sortBy$,
    (products, sortBy) => {
      const sorted = [...products];
      if (sortBy === 'name') {
        sorted.sort((a, b) => a.name.localeCompare(b.name));
      } else if (sortBy === 'price') {
        sorted.sort((a, b) => a.price - b.price);
      }
      return sorted;
    }
  );

  // Statistics
  readonly stats$ = this.select(
    this.products$,
    this.filteredProducts$,
    (all, filtered) => ({
      total: all.length,
      filtered: filtered.length,
      avgPrice: filtered.reduce((sum, p) => sum + p.price, 0) / filtered.length || 0,
    })
  );
}
```

### 3. Debounced Selectors

```typescript
@Injectable()
export class SearchStore extends ComponentStore<SearchState> {
  constructor() {
    super(initialState);
  }

  readonly query$ = this.select((state) => state.query);

  // Debounce search query
  readonly debouncedQuery$ = this.query$.pipe(
    debounceTime(300),
    distinctUntilChanged()
  );
}
```

## Effects

### 1. Basic Effects

```typescript
@Injectable()
export class UserStore extends ComponentStore<UserState> {
  constructor(private userService: UserService) {
    super(initialState);
  }

  // Effect to load user
  readonly loadUser = this.effect((userId$: Observable<string>) =>
    userId$.pipe(
      tap(() => this.setLoading(true)),
      switchMap((userId) =>
        this.userService.getUser(userId).pipe(
          tapResponse(
            (user) => this.setUser(user),
            (error: Error) => this.setError(error.message)
          )
        )
      )
    )
  );

  readonly setLoading = this.updater((state, loading: boolean) => ({
    ...state,
    loading,
  }));

  readonly setUser = this.updater((state, user: User) => ({
    ...state,
    user,
    loading: false,
    error: null,
  }));

  readonly setError = this.updater((state, error: string) => ({
    ...state,
    loading: false,
    error,
  }));
}

// Usage in component
ngOnInit() {
  this.userStore.loadUser(this.route.params.pipe(map(p => p['id'])));
}
```

### 2. Effects with Parameters

```typescript
@Injectable()
export class ProductStore extends ComponentStore<ProductState> {
  constructor(private productService: ProductService) {
    super(initialState);
  }

  // Effect that accepts trigger
  readonly searchProducts = this.effect((trigger$: Observable<string>) =>
    trigger$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => this.patchState({ loading: true })),
      switchMap((query) =>
        this.productService.search(query).pipe(
          tapResponse(
            (products) => this.patchState({ products, loading: false }),
            (error: Error) => this.patchState({ error: error.message, loading: false })
          )
        )
      )
    )
  );

  // Call with parameter
  search(query: string): void {
    this.searchProducts(of(query));
  }
}
```

### 3. Effects with State

```typescript
@Injectable()
export class CartStore extends ComponentStore<CartState> {
  constructor(private cartService: CartService) {
    super(initialState);
  }

  // Access state in effect
  readonly saveCart = this.effect((trigger$: Observable<void>) =>
    trigger$.pipe(
      withLatestFrom(this.items$),
      switchMap(([, items]) =>
        this.cartService.save(items).pipe(
          tapResponse(
            () => this.patchState({ lastSaved: new Date() }),
            (error: Error) => console.error('Save failed:', error)
          )
        )
      )
    )
  );

  readonly items$ = this.select((state) => state.items);
}
```

### 4. Chained Effects

```typescript
@Injectable()
export class OrderStore extends ComponentStore<OrderState> {
  constructor(
    private orderService: OrderService,
    private notificationService: NotificationService
  ) {
    super(initialState);
  }

  // First effect: create order
  readonly createOrder = this.effect((order$: Observable<Order>) =>
    order$.pipe(
      tap(() => this.patchState({ submitting: true })),
      switchMap((order) =>
        this.orderService.create(order).pipe(
          tapResponse(
            (createdOrder) => {
              this.patchState({ order: createdOrder, submitting: false });
              this.showSuccess('Order created successfully'); // Trigger second effect
            },
            (error: Error) => {
              this.patchState({ error: error.message, submitting: false });
              this.showError(error.message);
            }
          )
        )
      )
    )
  );

  // Second effect: show notification
  readonly showSuccess = this.effect((message$: Observable<string>) =>
    message$.pipe(
      tap((message) => this.notificationService.success(message))
    )
  );

  readonly showError = this.effect((message$: Observable<string>) =>
    message$.pipe(
      tap((message) => this.notificationService.error(message))
    )
  );
}
```

## Advanced Features

### 1. Lazy Initialization

```typescript
@Injectable()
export class ConfigStore extends ComponentStore<ConfigState> {
  constructor(private configService: ConfigService) {
    super(); // No initial state

    // Initialize lazily
    this.initializeState();
  }

  private initializeState(): void {
    this.setState({
      config: null,
      loaded: false,
    });

    // Or load from service
    this.loadConfig();
  }

  readonly loadConfig = this.effect((trigger$: Observable<void>) =>
    trigger$.pipe(
      switchMap(() =>
        this.configService.load().pipe(
          tapResponse(
            (config) => this.setState({ config, loaded: true }),
            (error) => console.error('Failed to load config:', error)
          )
        )
      )
    )
  );
}
```

### 2. Multiple Component Stores

```typescript
// Parent component
@Component({
  selector: 'app-dashboard',
  template: `
    <app-user-panel></app-user-panel>
    <app-stats-panel></app-stats-panel>
  `,
  providers: [UserStore, StatsStore], // Multiple stores
})
export class DashboardComponent {
  constructor(
    private userStore: UserStore,
    private statsStore: StatsStore
  ) {}
}

// Child components inject their respective stores
@Component({
  selector: 'app-user-panel',
  template: `...`,
})
export class UserPanelComponent {
  constructor(public userStore: UserStore) {}
}
```

### 3. Store Communication

```typescript
// Service to coordinate stores
@Injectable({ providedIn: 'root' })
export class AppCoordinator {
  constructor(
    private userStore: UserStore,
    private cartStore: CartStore
  ) {
    // When user changes, update cart
    this.userStore.user$
      .pipe(
        filter((user) => user !== null),
        switchMap((user) => this.cartStore.loadCart(of(user.id)))
      )
      .subscribe();
  }
}
```

### 4. Lifecycle Hooks

Component Store automatically manages subscription lifecycle:

```typescript
@Injectable()
export class DataStore extends ComponentStore<DataState> {
  constructor(private dataService: DataService) {
    super(initialState);

    // Auto-cleaned on destroy
    this.select((state) => state.filter)
      .pipe(
        debounceTime(300),
        distinctUntilChanged()
      )
      .subscribe((filter) => {
        this.loadData(of(filter));
      });
  }

  readonly loadData = this.effect((filter$: Observable<string>) =>
    filter$.pipe(
      switchMap((filter) => this.dataService.load(filter))
    )
  );
}

// Subscriptions automatically cleaned when component destroyed
```

## Integration

### 1. With NgRx Store

```typescript
@Injectable()
export class UserStore extends ComponentStore<UserState> {
  constructor(
    private store: Store,
    private userService: UserService
  ) {
    super(initialState);

    // Listen to global store
    this.store
      .select(selectCurrentUserId)
      .pipe(filter((id) => id !== null))
      .subscribe((id) => {
        this.loadUser(of(id));
      });
  }

  readonly loadUser = this.effect((id$: Observable<string>) =>
    id$.pipe(
      switchMap((id) => this.userService.getUser(id)),
      tapResponse(
        (user) => this.setUser(user),
        (error) => this.setError(error)
      )
    )
  );

  readonly setUser = this.updater((state, user: User) => ({
    ...state,
    user,
  }));

  readonly setError = this.updater((state, error: Error) => ({
    ...state,
    error: error.message,
  }));
}
```

### 2. With Forms

```typescript
@Injectable()
export class FormStore extends ComponentStore<FormState> {
  constructor() {
    super(initialState);
  }

  readonly updateFormValue = this.updater((state, value: any) => ({
    ...state,
    value,
    isDirty: true,
  }));

  readonly setValidationErrors = this.updater((state, errors: ValidationErrors | null) => ({
    ...state,
    errors,
    isValid: errors === null,
  }));

  // Connect to form
  connectForm(form: FormGroup): void {
    form.valueChanges
      .pipe(takeUntil(this.destroy$))
      .subscribe((value) => this.updateFormValue(value));

    form.statusChanges
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => this.setValidationErrors(form.errors));
  }
}
```

### 3. With Router

```typescript
@Injectable()
export class PageStore extends ComponentStore<PageState> {
  constructor(private route: ActivatedRoute) {
    super(initialState);

    // React to route params
    this.route.params
      .pipe(
        map((params) => params['id']),
        filter((id) => id !== undefined)
      )
      .subscribe((id) => {
        this.loadPage(of(id));
      });
  }

  readonly loadPage = this.effect((id$: Observable<string>) =>
    id$.pipe(
      switchMap((id) => this.pageService.load(id)),
      tapResponse(
        (page) => this.setPage(page),
        (error) => this.setError(error)
      )
    )
  );

  readonly setPage = this.updater((state, page: Page) => ({
    ...state,
    page,
  }));

  readonly setError = this.updater((state, error: Error) => ({
    ...state,
    error: error.message,
  }));
}
```

## Performance Optimization

### 1. Memoization

Selectors are automatically memoized:

```typescript
// Expensive computation only runs when inputs change
readonly expensiveComputation$ = this.select(
  this.items$,
  (items) => {
    console.log('Computing...'); // Only logs when items change
    return items.map((item) => ({
      ...item,
      computed: heavyComputation(item),
    }));
  }
);
```

### 2. OnPush Strategy

```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products$ | async">
      {{ product.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
  providers: [ProductStore],
})
export class ProductListComponent {
  products$ = this.productStore.sortedProducts$;

  constructor(private productStore: ProductStore) {}
}
```

### 3. Minimize State Size

```typescript
// ❌ Bad - storing derived data
interface BadState {
  items: Item[];
  filteredItems: Item[]; // Derived!
  itemCount: number; // Derived!
}

// ✅ Good - compute derived data
interface GoodState {
  items: Item[];
  filter: string;
}

readonly filteredItems$ = this.select(
  this.items$,
  this.filter$,
  (items, filter) => items.filter((i) => i.name.includes(filter))
);
```

## Common Mistakes

### 1. Not Providing Store

```typescript
// ❌ Wrong - no provider
@Component({
  selector: 'app-user',
  template: `...`,
})
export class UserComponent {
  constructor(private userStore: UserStore) {} // Error!
}

// ✅ Correct - provide at component level
@Component({
  selector: 'app-user',
  template: `...`,
  providers: [UserStore],
})
export class UserComponent {
  constructor(private userStore: UserStore) {}
}
```

### 2. Mutating State

```typescript
// ❌ Wrong - mutation
readonly addItem = this.updater((state, item: Item) => {
  state.items.push(item); // Mutation!
  return state;
});

// ✅ Correct - immutable
readonly addItem = this.updater((state, item: Item) => ({
  ...state,
  items: [...state.items, item],
}));
```

### 3. Subscribing Manually

```typescript
// ❌ Wrong - manual subscription
ngOnInit() {
  this.userStore.user$.subscribe((user) => {
    this.user = user; // Memory leak!
  });
}

// ✅ Correct - use async pipe
user$ = this.userStore.user$;

// template: {{ user$ | async }}
```

### 4. Not Handling Errors

```typescript
// ❌ Wrong - no error handling
readonly loadData = this.effect((trigger$: Observable<void>) =>
  trigger$.pipe(
    switchMap(() => this.service.load())
  )
);

// ✅ Correct - handle errors
readonly loadData = this.effect((trigger$: Observable<void>) =>
  trigger$.pipe(
    switchMap(() =>
      this.service.load().pipe(
        tapResponse(
          (data) => this.setData(data),
          (error: Error) => this.setError(error.message)
        )
      )
    )
  )
);
```

### 5. Complex Initial State

```typescript
// ❌ Wrong - complex initialization
constructor() {
  super({
    data: fetchDataSync(), // Side effect!
    timestamp: Date.now(),
  });
}

// ✅ Correct - simple initial state
constructor() {
  super(initialState);
  this.loadData(of(undefined)); // Load in effect
}
```

## Best Practices

### 1. Single Responsibility

```typescript
// Each store manages one concern
@Injectable()
export class UserStore extends ComponentStore<UserState> {}

@Injectable()
export class UserSettingsStore extends ComponentStore<SettingsState> {}

@Injectable()
export class UserPreferencesStore extends ComponentStore<PreferencesState> {}
```

### 2. Type-Safe Updates

```typescript
@Injectable()
export class TypedStore extends ComponentStore<TypedState> {
  readonly updateField = this.updater(
    <K extends keyof TypedState>(state: TypedState, payload: { key: K; value: TypedState[K] }) => ({
      ...state,
      [payload.key]: payload.value,
    })
  );
}

// Usage - fully typed
this.store.updateField({ key: 'loading', value: true });
```

### 3. Facade Pattern

```typescript
// Hide implementation details
@Injectable()
export class ProductFacade {
  // Expose only what's needed
  readonly products$ = this.productStore.sortedProducts$;
  readonly loading$ = this.productStore.loading$;
  readonly filter$ = this.productStore.filter$;

  constructor(private productStore: ProductStore) {}

  loadProducts(): void {
    this.productStore.loadProducts(of(undefined));
  }

  setFilter(filter: string): void {
    this.productStore.setFilter(filter);
  }
}
```

### 4. Effect Naming

```typescript
// Use descriptive names for effects
readonly loadUser = this.effect(...);
readonly saveUser = this.effect(...);
readonly deleteUser = this.effect(...);
readonly refreshUser = this.effect(...);
```

### 5. Document State Shape

```typescript
/**
 * User state interface
 * @property user - Current user data (null if not loaded)
 * @property loading - Loading indicator for async operations
 * @property error - Error message from failed operations
 * @property preferences - User preferences loaded separately
 */
export interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
  preferences: UserPreferences | null;
}
```

## When to Use

### Use Component Store When

1. **Local state** - Component or feature-specific state
2. **Simple state management** - Don't need global store complexity
3. **Isolated features** - State not shared across app
4. **Lazy-loaded modules** - State lifecycle matches module
5. **Multiple instances** - Need separate state per component instance

### Use NgRx Store Instead When

1. **Global state** - Shared across entire application
2. **Complex workflows** - Multiple effects and actions
3. **Time-travel debugging** - Need DevTools history
4. **State persistence** - Save/restore entire app state
5. **Large teams** - Need strict conventions

## Interview Questions

### 1. What is NgRx Component Store and when would you use it?

**Answer:** NgRx Component Store is a standalone state management solution for local component state. Use it when:
- State is local to a component or feature
- Don't need global state management complexity
- Want reactive state management with RxJS
- Need automatic lifecycle management
- Multiple component instances need separate state

Unlike NgRx Store, it's provided at component level and has simpler API.

### 2. How does Component Store differ from NgRx Store?

**Answer:**
- **Scope**: Component Store is local, NgRx Store is global
- **Lifecycle**: Component Store tied to component, Store is singleton
- **API**: Component Store simpler (no actions), Store has actions/reducers
- **DevTools**: Store has time-travel debugging, Component Store doesn't
- **Boilerplate**: Component Store less boilerplate
- **Use case**: Component Store for local state, Store for global state

### 3. Explain updaters in Component Store.

**Answer:** Updaters are functions that update state immutably:
```typescript
readonly setUser = this.updater((state, user: User) => ({
  ...state,
  user,
}));
```
- Pure functions that receive current state and payload
- Must return new state object (immutable)
- Can be called with values or observables
- Type-safe with TypeScript
- Automatically trigger selector updates

### 4. How do effects work in Component Store?

**Answer:** Effects handle side effects:
```typescript
readonly loadUser = this.effect((id$: Observable<string>) =>
  id$.pipe(
    switchMap((id) => this.service.getUser(id)),
    tapResponse(
      (user) => this.setUser(user),
      (error) => this.setError(error)
    )
  )
);
```
- Accept observable as input
- Return observable of side effects
- Use tapResponse for success/error handling
- Automatically manage subscriptions
- Can call updaters to update state

### 5. How do you combine multiple selectors?

**Answer:** Use select with multiple inputs:
```typescript
readonly filteredProducts$ = this.select(
  this.products$,
  this.filter$,
  this.sortBy$,
  (products, filter, sortBy) => {
    let result = products;
    if (filter) {
      result = result.filter(p => p.name.includes(filter));
    }
    if (sortBy === 'price') {
      result = [...result].sort((a, b) => a.price - b.price);
    }
    return result;
  }
);
```
Selectors are memoized and only recompute when inputs change.

## Key Takeaways

1. **Component Store is for local state** - scoped to component/feature, not global
2. **Provided at component level** - creates new instance per component
3. **Updaters modify state immutably** - pure functions that return new state
4. **Selectors compute derived state** - automatically memoized for performance
5. **Effects handle side effects** - async operations, API calls, etc.
6. **Automatic lifecycle management** - subscriptions cleaned up on destroy
7. **RxJS-based** - leverages observables for reactive programming
8. **Less boilerplate than NgRx Store** - no actions, simpler API
9. **Can access state imperatively** - use get() when needed
10. **Type-safe with TypeScript** - full type inference and checking

## Resources

### Official Documentation
- [Component Store Docs](https://ngrx.io/guide/component-store)
- [API Reference](https://ngrx.io/guide/component-store/api)
- [Examples](https://ngrx.io/guide/component-store/usage)

### Tutorials
- [Component Store Introduction](https://ngrx.io/guide/component-store)
- [Alex Okrushko's Blog](https://blog.nrwl.io/ngrx-component-store-is-here-9d2d5a53f61f)

### Community
- [GitHub Repository](https://github.com/ngrx/platform)
- [Discord Community](https://discord.com/invite/ngrx)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/ngrx)
