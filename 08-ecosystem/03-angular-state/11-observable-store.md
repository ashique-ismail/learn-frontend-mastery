# Observable Store Pattern

## The Idea

**In plain English:** An Observable Store is a central place in your app that holds data (like who is logged in, or what is in a shopping cart) and automatically notifies every part of the app that cares whenever that data changes — no manual refreshing needed.

**Real-world analogy:** Think of a school noticeboard where the office pins up important announcements. Any student who has signed up to watch the noticeboard instantly sees new updates the moment they are posted.

- The noticeboard = the store (holds the current state)
- An announcement pinned to it = a state update
- A student who signed up to watch = a component subscribed to an observable
- The act of a new update appearing automatically = the observable emitting a new value

---

## Overview

The Observable Store pattern is a lightweight state management approach in Angular that leverages RxJS observables and services to create centralized, reactive state containers. Unlike full-featured state management libraries, Observable Store provides a simple, flexible pattern for managing application state using Angular's built-in dependency injection and RxJS capabilities.

This pattern is particularly well-suited for small to medium-sized applications where the overhead of libraries like NgRx might be unnecessary, but you still want reactive, predictable state management.

## Installation and Setup

### Basic Service-Based Observable Store

No external dependencies required - uses Angular and RxJS:

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

interface AppState {
  user: User | null;
  isLoading: boolean;
  error: string | null;
}

interface User {
  id: string;
  name: string;
  email: string;
}

@Injectable({
  providedIn: 'root'
})
export class StoreService {
  private readonly _state$ = new BehaviorSubject<AppState>({
    user: null,
    isLoading: false,
    error: null
  });

  // Expose the state as an observable (read-only)
  readonly state$ = this._state$.asObservable();

  // Get current state snapshot
  get state(): AppState {
    return this._state$.getValue();
  }

  // Update state immutably
  private setState(newState: Partial<AppState>): void {
    this._state$.next({
      ...this.state,
      ...newState
    });
  }

  // Selectors for specific state slices
  selectUser(): Observable<User | null> {
    return this.state$.pipe(
      map(state => state.user),
      distinctUntilChanged()
    );
  }

  selectIsLoading(): Observable<boolean> {
    return this.state$.pipe(
      map(state => state.isLoading),
      distinctUntilChanged()
    );
  }

  selectError(): Observable<string | null> {
    return this.state$.pipe(
      map(state => state.error),
      distinctUntilChanged()
    );
  }

  // Actions to modify state
  setUser(user: User): void {
    this.setState({ user, error: null });
  }

  setLoading(isLoading: boolean): void {
    this.setState({ isLoading });
  }

  setError(error: string): void {
    this.setState({ error, isLoading: false });
  }

  clearError(): void {
    this.setState({ error: null });
  }

  logout(): void {
    this.setState({ user: null, error: null });
  }
}
```

### Using Observable Store Library

Install the `@codewithdan/observable-store` package:

```bash
npm install @codewithdan/observable-store
```

```typescript
import { Injectable } from '@angular/core';
import { ObservableStore } from '@codewithdan/observable-store';

interface CustomersState {
  customers: Customer[];
  selectedCustomer: Customer | null;
}

interface Customer {
  id: number;
  name: string;
  city: string;
}

@Injectable({
  providedIn: 'root'
})
export class CustomersStore extends ObservableStore<CustomersState> {
  constructor() {
    super({ trackStateHistory: true, logStateChanges: true });
    this.setState({ customers: [], selectedCustomer: null }, 'INIT_STATE');
  }

  addCustomer(customer: Customer): void {
    const customers = [...this.getState().customers, customer];
    this.setState({ customers }, 'ADD_CUSTOMER');
  }

  removeCustomer(id: number): void {
    const customers = this.getState().customers.filter(c => c.id !== id);
    this.setState({ customers }, 'REMOVE_CUSTOMER');
  }

  selectCustomer(customer: Customer): void {
    this.setState({ selectedCustomer: customer }, 'SELECT_CUSTOMER');
  }

  updateCustomer(customer: Customer): void {
    const customers = this.getState().customers.map(c =>
      c.id === customer.id ? customer : c
    );
    this.setState({ customers }, 'UPDATE_CUSTOMER');
  }
}
```

## Core Concepts

### 1. BehaviorSubject as State Container

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
}

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class TodoStore {
  private readonly initialState: TodoState = {
    todos: [],
    filter: 'all'
  };

  private readonly _state$ = new BehaviorSubject<TodoState>(this.initialState);
  
  // Public observable
  readonly state$: Observable<TodoState> = this._state$.asObservable();

  get snapshot(): TodoState {
    return this._state$.getValue();
  }

  private updateState(partial: Partial<TodoState>): void {
    const current = this.snapshot;
    this._state$.next({ ...current, ...partial });
  }

  addTodo(text: string): void {
    const newTodo: Todo = {
      id: Date.now().toString(),
      text,
      completed: false
    };
    
    this.updateState({
      todos: [...this.snapshot.todos, newTodo]
    });
  }

  toggleTodo(id: string): void {
    const todos = this.snapshot.todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    );
    this.updateState({ todos });
  }

  removeTodo(id: string): void {
    const todos = this.snapshot.todos.filter(todo => todo.id !== id);
    this.updateState({ todos });
  }

  setFilter(filter: TodoState['filter']): void {
    this.updateState({ filter });
  }

  clearCompleted(): void {
    const todos = this.snapshot.todos.filter(todo => !todo.completed);
    this.updateState({ todos });
  }
}
```

### 2. Selectors and Derived State

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, combineLatest } from 'rxjs';
import { map, distinctUntilChanged, shareReplay } from 'rxjs/operators';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

interface ProductState {
  products: Product[];
  searchTerm: string;
  selectedCategory: string | null;
  sortBy: 'name' | 'price';
}

@Injectable({
  providedIn: 'root'
})
export class ProductStore {
  private readonly _state$ = new BehaviorSubject<ProductState>({
    products: [],
    searchTerm: '',
    selectedCategory: null,
    sortBy: 'name'
  });

  readonly state$ = this._state$.asObservable();

  // Simple selectors
  readonly products$ = this.select(state => state.products);
  readonly searchTerm$ = this.select(state => state.searchTerm);
  readonly selectedCategory$ = this.select(state => state.selectedCategory);
  readonly sortBy$ = this.select(state => state.sortBy);

  // Derived selectors with memoization
  readonly filteredProducts$ = combineLatest([
    this.products$,
    this.searchTerm$,
    this.selectedCategory$
  ]).pipe(
    map(([products, searchTerm, category]) => {
      let filtered = products;

      if (searchTerm) {
        const term = searchTerm.toLowerCase();
        filtered = filtered.filter(p =>
          p.name.toLowerCase().includes(term)
        );
      }

      if (category) {
        filtered = filtered.filter(p => p.category === category);
      }

      return filtered;
    }),
    shareReplay(1)
  );

  readonly sortedProducts$ = combineLatest([
    this.filteredProducts$,
    this.sortBy$
  ]).pipe(
    map(([products, sortBy]) => {
      const sorted = [...products];
      if (sortBy === 'name') {
        sorted.sort((a, b) => a.name.localeCompare(b.name));
      } else {
        sorted.sort((a, b) => a.price - b.price);
      }
      return sorted;
    }),
    shareReplay(1)
  );

  readonly availableCategories$ = this.products$.pipe(
    map(products => {
      const categories = new Set(products.map(p => p.category));
      return Array.from(categories).sort();
    }),
    shareReplay(1)
  );

  readonly inStockCount$ = this.products$.pipe(
    map(products => products.filter(p => p.inStock).length)
  );

  readonly totalValue$ = this.products$.pipe(
    map(products => products.reduce((sum, p) => sum + p.price, 0))
  );

  private select<T>(selector: (state: ProductState) => T): Observable<T> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged(),
      shareReplay(1)
    );
  }

  private get state(): ProductState {
    return this._state$.getValue();
  }

  private setState(partial: Partial<ProductState>): void {
    this._state$.next({ ...this.state, ...partial });
  }

  setProducts(products: Product[]): void {
    this.setState({ products });
  }

  setSearchTerm(searchTerm: string): void {
    this.setState({ searchTerm });
  }

  setCategory(selectedCategory: string | null): void {
    this.setState({ selectedCategory });
  }

  setSortBy(sortBy: ProductState['sortBy']): void {
    this.setState({ sortBy });
  }
}
```

### 3. Global vs Local Store

**Global Store (Singleton):**

```typescript
// Global store - provided in root
@Injectable({
  providedIn: 'root'
})
export class AuthStore {
  private readonly _state$ = new BehaviorSubject<AuthState>({
    user: null,
    token: null,
    isAuthenticated: false
  });

  readonly state$ = this._state$.asObservable();
  readonly user$ = this.state$.pipe(map(s => s.user), distinctUntilChanged());
  readonly isAuthenticated$ = this.state$.pipe(
    map(s => s.isAuthenticated),
    distinctUntilChanged()
  );

  login(user: User, token: string): void {
    this._state$.next({
      user,
      token,
      isAuthenticated: true
    });
  }

  logout(): void {
    this._state$.next({
      user: null,
      token: null,
      isAuthenticated: false
    });
  }
}
```

**Local Store (Component-Level):**

```typescript
// Local store - provided in component
interface FormState {
  data: Record<string, any>;
  isDirty: boolean;
  isValid: boolean;
  errors: Record<string, string>;
}

@Injectable()
export class FormStore {
  private readonly _state$ = new BehaviorSubject<FormState>({
    data: {},
    isDirty: false,
    isValid: true,
    errors: {}
  });

  readonly state$ = this._state$.asObservable();
  readonly isDirty$ = this.state$.pipe(map(s => s.isDirty));
  readonly isValid$ = this.state$.pipe(map(s => s.isValid));
  readonly errors$ = this.state$.pipe(map(s => s.errors));

  updateField(field: string, value: any): void {
    const current = this._state$.getValue();
    this._state$.next({
      ...current,
      data: { ...current.data, [field]: value },
      isDirty: true
    });
  }

  setErrors(errors: Record<string, string>): void {
    const current = this._state$.getValue();
    this._state$.next({
      ...current,
      errors,
      isValid: Object.keys(errors).length === 0
    });
  }

  reset(): void {
    this._state$.next({
      data: {},
      isDirty: false,
      isValid: true,
      errors: {}
    });
  }
}

@Component({
  selector: 'app-user-form',
  template: `
    <form>
      <div *ngIf="isDirty$ | async">Unsaved changes</div>
      <div *ngIf="!(isValid$ | async)">Form has errors</div>
    </form>
  `,
  providers: [FormStore] // Local instance per component
})
export class UserFormComponent {
  readonly isDirty$ = this.formStore.isDirty$;
  readonly isValid$ = this.formStore.isValid$;

  constructor(private formStore: FormStore) {}
}
```

### 4. Async Actions with Effects

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { tap, catchError, finalize } from 'rxjs/operators';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserState {
  users: User[];
  selectedUser: User | null;
  loading: boolean;
  error: string | null;
}

@Injectable({
  providedIn: 'root'
})
export class UserStore {
  private readonly _state$ = new BehaviorSubject<UserState>({
    users: [],
    selectedUser: null,
    loading: false,
    error: null
  });

  readonly state$ = this._state$.asObservable();
  readonly users$ = this.state$.pipe(map(s => s.users));
  readonly selectedUser$ = this.state$.pipe(map(s => s.selectedUser));
  readonly loading$ = this.state$.pipe(map(s => s.loading));
  readonly error$ = this.state$.pipe(map(s => s.error));

  constructor(private http: HttpClient) {}

  private get state(): UserState {
    return this._state$.getValue();
  }

  private setState(partial: Partial<UserState>): void {
    this._state$.next({ ...this.state, ...partial });
  }

  // Async action: Load users
  loadUsers(): Observable<User[]> {
    this.setState({ loading: true, error: null });

    return this.http.get<User[]>('/api/users').pipe(
      tap(users => {
        this.setState({ users, loading: false });
      }),
      catchError(error => {
        this.setState({
          error: error.message,
          loading: false
        });
        return throwError(() => error);
      })
    );
  }

  // Async action: Load single user
  loadUser(id: string): Observable<User> {
    this.setState({ loading: true, error: null });

    return this.http.get<User>(`/api/users/${id}`).pipe(
      tap(user => {
        this.setState({ selectedUser: user, loading: false });
      }),
      catchError(error => {
        this.setState({
          error: error.message,
          loading: false
        });
        return throwError(() => error);
      })
    );
  }

  // Async action: Create user
  createUser(user: Omit<User, 'id'>): Observable<User> {
    this.setState({ loading: true, error: null });

    return this.http.post<User>('/api/users', user).pipe(
      tap(newUser => {
        this.setState({
          users: [...this.state.users, newUser],
          loading: false
        });
      }),
      catchError(error => {
        this.setState({
          error: error.message,
          loading: false
        });
        return throwError(() => error);
      })
    );
  }

  // Async action: Update user
  updateUser(id: string, updates: Partial<User>): Observable<User> {
    this.setState({ loading: true, error: null });

    return this.http.patch<User>(`/api/users/${id}`, updates).pipe(
      tap(updatedUser => {
        const users = this.state.users.map(u =>
          u.id === id ? updatedUser : u
        );
        this.setState({ users, loading: false });
      }),
      catchError(error => {
        this.setState({
          error: error.message,
          loading: false
        });
        return throwError(() => error);
      })
    );
  }

  // Async action: Delete user
  deleteUser(id: string): Observable<void> {
    this.setState({ loading: true, error: null });

    return this.http.delete<void>(`/api/users/${id}`).pipe(
      tap(() => {
        const users = this.state.users.filter(u => u.id !== id);
        this.setState({ users, loading: false });
      }),
      catchError(error => {
        this.setState({
          error: error.message,
          loading: false
        });
        return throwError(() => error);
      })
    );
  }

  clearError(): void {
    this.setState({ error: null });
  }
}
```

### 5. Computed Values with RxJS

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, combineLatest, Observable } from 'rxjs';
import { map, shareReplay, distinctUntilChanged } from 'rxjs/operators';

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  discountCode: string | null;
  taxRate: number;
}

@Injectable({
  providedIn: 'root'
})
export class CartStore {
  private readonly _state$ = new BehaviorSubject<CartState>({
    items: [],
    discountCode: null,
    taxRate: 0.08
  });

  readonly state$ = this._state$.asObservable();

  // Simple selectors
  readonly items$ = this.state$.pipe(
    map(s => s.items),
    distinctUntilChanged()
  );

  readonly discountCode$ = this.state$.pipe(
    map(s => s.discountCode),
    distinctUntilChanged()
  );

  readonly taxRate$ = this.state$.pipe(
    map(s => s.taxRate),
    distinctUntilChanged()
  );

  // Computed values
  readonly itemCount$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + item.quantity, 0)),
    shareReplay(1)
  );

  readonly subtotal$ = this.items$.pipe(
    map(items =>
      items.reduce((sum, item) => sum + item.price * item.quantity, 0)
    ),
    shareReplay(1)
  );

  readonly discount$ = combineLatest([
    this.subtotal$,
    this.discountCode$
  ]).pipe(
    map(([subtotal, code]) => {
      if (!code) return 0;
      // Example discount logic
      if (code === 'SAVE10') return subtotal * 0.1;
      if (code === 'SAVE20') return subtotal * 0.2;
      return 0;
    }),
    shareReplay(1)
  );

  readonly tax$ = combineLatest([
    this.subtotal$,
    this.discount$,
    this.taxRate$
  ]).pipe(
    map(([subtotal, discount, taxRate]) =>
      (subtotal - discount) * taxRate
    ),
    shareReplay(1)
  );

  readonly total$ = combineLatest([
    this.subtotal$,
    this.discount$,
    this.tax$
  ]).pipe(
    map(([subtotal, discount, tax]) => subtotal - discount + tax),
    shareReplay(1)
  );

  readonly isEmpty$ = this.items$.pipe(
    map(items => items.length === 0),
    distinctUntilChanged()
  );

  readonly summary$ = combineLatest([
    this.itemCount$,
    this.subtotal$,
    this.discount$,
    this.tax$,
    this.total$
  ]).pipe(
    map(([itemCount, subtotal, discount, tax, total]) => ({
      itemCount,
      subtotal,
      discount,
      tax,
      total
    })),
    shareReplay(1)
  );

  private get state(): CartState {
    return this._state$.getValue();
  }

  private setState(partial: Partial<CartState>): void {
    this._state$.next({ ...this.state, ...partial });
  }

  addItem(item: Omit<CartItem, 'quantity'>): void {
    const items = [...this.state.items];
    const existingIndex = items.findIndex(i => i.productId === item.productId);

    if (existingIndex >= 0) {
      items[existingIndex] = {
        ...items[existingIndex],
        quantity: items[existingIndex].quantity + 1
      };
    } else {
      items.push({ ...item, quantity: 1 });
    }

    this.setState({ items });
  }

  removeItem(productId: string): void {
    const items = this.state.items.filter(i => i.productId !== productId);
    this.setState({ items });
  }

  updateQuantity(productId: string, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    const items = this.state.items.map(item =>
      item.productId === productId ? { ...item, quantity } : item
    );
    this.setState({ items });
  }

  applyDiscountCode(code: string): void {
    this.setState({ discountCode: code });
  }

  clearDiscountCode(): void {
    this.setState({ discountCode: null });
  }

  clearCart(): void {
    this.setState({ items: [], discountCode: null });
  }
}
```

## Component Integration

### Using Observable Store in Components

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="filters">
      <input
        type="text"
        [value]="searchTerm$ | async"
        (input)="onSearchChange($event)"
        placeholder="Search products..."
      />
      
      <select (change)="onCategoryChange($event)">
        <option value="">All Categories</option>
        <option
          *ngFor="let category of categories$ | async"
          [value]="category"
        >
          {{ category }}
        </option>
      </select>
      
      <select (change)="onSortChange($event)">
        <option value="name">Sort by Name</option>
        <option value="price">Sort by Price</option>
      </select>
    </div>

    <div class="product-grid">
      <div
        *ngFor="let product of sortedProducts$ | async"
        class="product-card"
      >
        <h3>{{ product.name }}</h3>
        <p>{{ product.price | currency }}</p>
        <span [class.in-stock]="product.inStock">
          {{ product.inStock ? 'In Stock' : 'Out of Stock' }}
        </span>
      </div>
    </div>

    <div class="stats">
      <p>In Stock: {{ inStockCount$ | async }}</p>
      <p>Total Value: {{ totalValue$ | async | currency }}</p>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  sortedProducts$ = this.productStore.sortedProducts$;
  categories$ = this.productStore.availableCategories$;
  searchTerm$ = this.productStore.searchTerm$;
  inStockCount$ = this.productStore.inStockCount$;
  totalValue$ = this.productStore.totalValue$;

  constructor(private productStore: ProductStore) {}

  ngOnInit(): void {
    // Load initial data
    this.loadProducts();
  }

  onSearchChange(event: Event): void {
    const term = (event.target as HTMLInputElement).value;
    this.productStore.setSearchTerm(term);
  }

  onCategoryChange(event: Event): void {
    const category = (event.target as HTMLSelectElement).value || null;
    this.productStore.setCategory(category);
  }

  onSortChange(event: Event): void {
    const sortBy = (event.target as HTMLSelectElement).value as 'name' | 'price';
    this.productStore.setSortBy(sortBy);
  }

  private loadProducts(): void {
    // In real app, this would load from API
    const products = [
      { id: '1', name: 'Product 1', price: 29.99, category: 'Electronics', inStock: true },
      { id: '2', name: 'Product 2', price: 49.99, category: 'Books', inStock: false },
      // ... more products
    ];
    this.productStore.setProducts(products);
  }
}
```

### Handling Async Operations

```typescript
@Component({
  selector: 'app-user-dashboard',
  template: `
    <div *ngIf="loading$ | async" class="loading">
      Loading users...
    </div>

    <div *ngIf="error$ | async as error" class="error">
      {{ error }}
      <button (click)="retry()">Retry</button>
    </div>

    <div *ngIf="users$ | async as users" class="user-list">
      <div *ngFor="let user of users" class="user-card">
        <h3>{{ user.name }}</h3>
        <p>{{ user.email }}</p>
        <button (click)="editUser(user)">Edit</button>
        <button (click)="deleteUser(user.id)">Delete</button>
      </div>
    </div>

    <button (click)="addUser()">Add User</button>
  `
})
export class UserDashboardComponent implements OnInit {
  users$ = this.userStore.users$;
  loading$ = this.userStore.loading$;
  error$ = this.userStore.error$;

  constructor(private userStore: UserStore) {}

  ngOnInit(): void {
    this.loadUsers();
  }

  loadUsers(): void {
    this.userStore.loadUsers().subscribe();
  }

  addUser(): void {
    const newUser = {
      name: 'New User',
      email: 'new@example.com'
    };

    this.userStore.createUser(newUser).subscribe({
      next: () => console.log('User created'),
      error: (err) => console.error('Failed to create user', err)
    });
  }

  editUser(user: User): void {
    const updates = { name: 'Updated Name' };
    
    this.userStore.updateUser(user.id, updates).subscribe({
      next: () => console.log('User updated'),
      error: (err) => console.error('Failed to update user', err)
    });
  }

  deleteUser(id: string): void {
    if (!confirm('Are you sure?')) return;

    this.userStore.deleteUser(id).subscribe({
      next: () => console.log('User deleted'),
      error: (err) => console.error('Failed to delete user', err)
    });
  }

  retry(): void {
    this.userStore.clearError();
    this.loadUsers();
  }
}
```

## Advanced Patterns

### Store with History/Undo

```typescript
interface HistoryState<T> {
  past: T[];
  present: T;
  future: T[];
}

@Injectable({
  providedIn: 'root'
})
export class EditorStore {
  private readonly _history$ = new BehaviorSubject<HistoryState<string>>({
    past: [],
    present: '',
    future: []
  });

  readonly history$ = this._history$.asObservable();
  readonly content$ = this.history$.pipe(map(h => h.present));
  readonly canUndo$ = this.history$.pipe(map(h => h.past.length > 0));
  readonly canRedo$ = this.history$.pipe(map(h => h.future.length > 0));

  private get history(): HistoryState<string> {
    return this._history$.getValue();
  }

  updateContent(content: string): void {
    const { past, present } = this.history;
    
    this._history$.next({
      past: [...past, present],
      present: content,
      future: [] // Clear future on new change
    });
  }

  undo(): void {
    const { past, present, future } = this.history;
    if (past.length === 0) return;

    const previous = past[past.length - 1];
    const newPast = past.slice(0, past.length - 1);

    this._history$.next({
      past: newPast,
      present: previous,
      future: [present, ...future]
    });
  }

  redo(): void {
    const { past, present, future } = this.history;
    if (future.length === 0) return;

    const next = future[0];
    const newFuture = future.slice(1);

    this._history$.next({
      past: [...past, present],
      present: next,
      future: newFuture
    });
  }

  reset(): void {
    this._history$.next({
      past: [],
      present: '',
      future: []
    });
  }
}
```

### Store with Persistence

```typescript
@Injectable({
  providedIn: 'root'
})
export class PersistedStore {
  private readonly STORAGE_KEY = 'app_state';
  
  private readonly _state$ = new BehaviorSubject<AppState>(
    this.loadFromStorage()
  );

  readonly state$ = this._state$.asObservable();

  constructor() {
    // Auto-save to localStorage on state changes
    this.state$.subscribe(state => {
      this.saveToStorage(state);
    });
  }

  private loadFromStorage(): AppState {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      return stored ? JSON.parse(stored) : this.getDefaultState();
    } catch (error) {
      console.error('Failed to load state from storage', error);
      return this.getDefaultState();
    }
  }

  private saveToStorage(state: AppState): void {
    try {
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(state));
    } catch (error) {
      console.error('Failed to save state to storage', error);
    }
  }

  private getDefaultState(): AppState {
    return {
      user: null,
      preferences: {
        theme: 'light',
        language: 'en'
      }
    };
  }

  clearStorage(): void {
    localStorage.removeItem(this.STORAGE_KEY);
    this._state$.next(this.getDefaultState());
  }
}
```

## Common Mistakes

1. **Mutating state directly**: Always create new objects/arrays instead of mutating existing state.

2. **Not using distinctUntilChanged**: Selectors may emit unnecessarily without this operator.

3. **Memory leaks**: Not unsubscribing from observables in components (use async pipe or takeUntil).

4. **Synchronous getValue() in templates**: Use async pipe instead of calling getValue() in templates.

5. **Over-normalizing small stores**: Observable Store works best with simpler, denormalized state.

## Best Practices

1. **Immutable updates**: Always spread state objects when updating
2. **Type safety**: Use TypeScript interfaces for all state shapes
3. **Selective exposure**: Only expose necessary observables, keep BehaviorSubject private
4. **Memoization**: Use shareReplay(1) for expensive computed values
5. **Single responsibility**: One store per feature domain
6. **Async pipe preference**: Use async pipe in templates instead of manual subscriptions
7. **Error handling**: Include loading and error states for async operations
8. **Testing**: Store services are easy to unit test with mock data

## When to Use Observable Store

**Use Observable Store when:**

- Small to medium-sized applications
- Simple state management needs
- Team unfamiliar with complex state libraries
- Want lightweight, flexible approach
- RxJS expertise available

**Consider alternatives when:**

- Large-scale applications with complex state
- Need time-travel debugging
- Want devtools integration
- Multiple developers need strict patterns
- Complex entity relationships

## Interview Questions

1. **Q: How does Observable Store differ from NgRx?**
   A: Observable Store is a lightweight pattern using services and RxJS, while NgRx is a full Redux implementation with actions, reducers, effects, and devtools integration. Observable Store has less boilerplate but fewer architectural constraints and tooling.

2. **Q: Why use BehaviorSubject instead of Subject?**
   A: BehaviorSubject stores the current value and emits it immediately to new subscribers, making it ideal for state management. It also allows synchronous access via getValue() for current state snapshots.

3. **Q: How do you prevent memory leaks with Observable Store?**
   A: Use the async pipe in templates (auto-unsubscribes), use takeUntil with a destroy subject in imperative code, or use Angular's DestroyRef with takeUntilDestroyed in Angular 16+.

4. **Q: What's the difference between global and local stores?**
   A: Global stores are singletons (providedIn: 'root') shared across the app, while local stores are provided in components/modules and create separate instances per provider scope.

5. **Q: How do you handle derived/computed state?**
   A: Use RxJS operators like map, combineLatest, and shareReplay to create derived observables from base state selectors. This provides memoization and only recalculates when dependencies change.

6. **Q: Should you use getValue() in components?**
   A: Generally no - prefer observables and the async pipe for reactivity. Only use getValue() internally within the store for state updates or in rare cases where synchronous access is required.

## Key Takeaways

1. Observable Store leverages Angular services and RxJS for state management
2. BehaviorSubject is the core primitive for storing and emitting state
3. Selectors use map and distinctUntilChanged for optimized subscriptions
4. Immutable updates prevent bugs and enable change detection optimization
5. Derived state uses combineLatest and shareReplay for memoization
6. Global stores are singletons; local stores are component-scoped
7. Async operations should include loading and error states
8. The async pipe prevents memory leaks and simplifies templates
9. Observable Store offers flexibility but requires discipline
10. Best suited for small-to-medium apps where Redux patterns are overkill

## Resources

- [Observable Store Library](https://github.com/DanWahlin/Observable-Store)
- [Angular Services as State Management](https://blog.angular-university.io/how-to-build-angular2-apps-using-rxjs-observable-data-services-pitfalls-to-avoid/)
- [RxJS State Management Patterns](https://www.bitovi.com/blog/rxjs-state-management-patterns)
- [Building Custom State Management](https://indepth.dev/posts/1523/building-a-simple-state-management-system-in-angular)
- [BehaviorSubject Best Practices](https://blog.angular-university.io/rxjs-subjects-and-multicasting/)
