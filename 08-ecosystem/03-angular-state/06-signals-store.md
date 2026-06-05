# Angular Signals Store

## The Idea

**In plain English:** A Signals Store is a way to keep track of information (called "state") in your app so that whenever that information changes, only the parts of the screen that use it automatically update — no manual refresh needed. A "signal" is just a special variable that the app watches, and a "store" is the place where you organize and keep those watched variables together.

**Real-world analogy:** Think of a live scoreboard at a sports stadium. The scoreboard shows the current score, time remaining, and player stats. Whenever something changes — a goal is scored, time ticks down — only the relevant section of the board updates instantly, without reprinting the whole thing.

- The scoreboard = the Signals Store (the central place holding all the current data)
- Each individual section of the board (score, timer, player name) = a signal (a single tracked piece of data)
- The sections that auto-recalculate from others, like "points per minute" = computed signals (values derived automatically from other signals)

---

## Overview

Angular Signals represent a fundamental shift in how Angular handles reactivity and state management. Introduced in Angular 16 and matured in Angular 17+, Signals provide a fine-grained reactivity system that enables automatic change detection without Zone.js. The Signals Store pattern leverages this primitive to create lightweight, performant state management solutions with computed values, effects, and seamless integration with Angular's template system.

Unlike traditional Observable-based state management, Signals are synchronous, pull-based, and offer better performance characteristics for local state management. When combined with Angular's upcoming @ngrx/signals package, they provide a powerful alternative to complex state management libraries.

## Installation and Setup

### Built-in Angular Signals (Angular 16+)

```bash
# Ensure Angular 16+ is installed
npm install @angular/core@latest
```

### Basic Signal Store Service

```typescript
import { Injectable, signal, computed, effect } from '@angular/core';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserState {
  users: User[];
  selectedUserId: string | null;
  loading: boolean;
  error: string | null;
}

@Injectable({
  providedIn: 'root'
})
export class UserSignalsStore {
  // Private writable signals
  private readonly _state = signal<UserState>({
    users: [],
    selectedUserId: null,
    loading: false,
    error: null
  });

  // Public read-only signals
  readonly users = computed(() => this._state().users);
  readonly selectedUserId = computed(() => this._state().selectedUserId);
  readonly loading = computed(() => this._state().loading);
  readonly error = computed(() => this._state().error);

  // Computed derived state
  readonly selectedUser = computed(() => {
    const state = this._state();
    return state.users.find(u => u.id === state.selectedUserId) ?? null;
  });

  readonly userCount = computed(() => this._state().users.length);

  constructor() {
    // Effects run when signals change
    effect(() => {
      console.log('User count changed:', this.userCount());
    });
  }

  // Actions to update state
  setUsers(users: User[]): void {
    this._state.update(state => ({
      ...state,
      users,
      error: null
    }));
  }

  selectUser(userId: string | null): void {
    this._state.update(state => ({
      ...state,
      selectedUserId: userId
    }));
  }

  setLoading(loading: boolean): void {
    this._state.update(state => ({ ...state, loading }));
  }

  setError(error: string): void {
    this._state.update(state => ({
      ...state,
      error,
      loading: false
    }));
  }

  addUser(user: User): void {
    this._state.update(state => ({
      ...state,
      users: [...state.users, user]
    }));
  }

  updateUser(id: string, updates: Partial<User>): void {
    this._state.update(state => ({
      ...state,
      users: state.users.map(u =>
        u.id === id ? { ...u, ...updates } : u
      )
    }));
  }

  removeUser(id: string): void {
    this._state.update(state => ({
      ...state,
      users: state.users.filter(u => u.id !== id),
      selectedUserId: state.selectedUserId === id ? null : state.selectedUserId
    }));
  }

  reset(): void {
    this._state.set({
      users: [],
      selectedUserId: null,
      loading: false,
      error: null
    });
  }
}
```

### NgRx SignalStore (Experimental)

```bash
npm install @ngrx/signals
```

```typescript
import { signalStore, withState, withComputed, withMethods } from '@ngrx/signals';
import { computed } from '@angular/core';

interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

interface ProductState {
  products: Product[];
  filter: string;
  loading: boolean;
}

export const ProductStore = signalStore(
  { providedIn: 'root' },
  withState<ProductState>({
    products: [],
    filter: '',
    loading: false
  }),
  withComputed(({ products, filter }) => ({
    filteredProducts: computed(() => {
      const filterLower = filter().toLowerCase();
      return products().filter(p =>
        p.name.toLowerCase().includes(filterLower)
      );
    }),
    inStockCount: computed(() =>
      products().filter(p => p.inStock).length
    ),
    totalValue: computed(() =>
      products().reduce((sum, p) => sum + p.price, 0)
    )
  })),
  withMethods((store) => ({
    setProducts(products: Product[]): void {
      store.products = products;
    },
    setFilter(filter: string): void {
      store.filter = filter;
    },
    setLoading(loading: boolean): void {
      store.loading = loading;
    },
    addProduct(product: Product): void {
      store.products = [...store.products, product];
    },
    updateProduct(id: string, updates: Partial<Product>): void {
      store.products = store.products.map(p =>
        p.id === id ? { ...p, ...updates } : p
      );
    },
    removeProduct(id: string): void {
      store.products = store.products.filter(p => p.id !== id);
    }
  }))
);
```

## Core Concepts

### 1. Signals Basics

```typescript
import { Component, signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <p>Double: {{ double() }}</p>
      <button (click)="increment()">Increment</button>
      <button (click)="reset()">Reset</button>
    </div>
  `
})
export class CounterComponent {
  // Writable signal
  count = signal(0);

  // Computed signal (automatically updates when count changes)
  double = computed(() => this.count() * 2);

  constructor() {
    // Effect runs when count changes
    effect(() => {
      console.log('Count changed to:', this.count());
    });
  }

  increment(): void {
    // Update with new value
    this.count.set(this.count() + 1);
    
    // Or use update for transformations
    // this.count.update(n => n + 1);
  }

  reset(): void {
    this.count.set(0);
  }
}
```

### 2. Computed Signals

```typescript
import { Injectable, signal, computed } from '@angular/core';

interface Task {
  id: string;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  dueDate: Date;
}

@Injectable({
  providedIn: 'root'
})
export class TaskStore {
  private readonly _tasks = signal<Task[]>([]);
  private readonly _filter = signal<'all' | 'active' | 'completed'>('all');
  private readonly _sortBy = signal<'title' | 'priority' | 'dueDate'>('dueDate');

  // Public read-only signals
  readonly tasks = this._tasks.asReadonly();
  readonly filter = this._filter.asReadonly();
  readonly sortBy = this._sortBy.asReadonly();

  // Computed: filtered tasks
  readonly filteredTasks = computed(() => {
    const tasks = this._tasks();
    const filter = this._filter();

    if (filter === 'active') {
      return tasks.filter(t => !t.completed);
    }
    if (filter === 'completed') {
      return tasks.filter(t => t.completed);
    }
    return tasks;
  });

  // Computed: sorted tasks
  readonly sortedTasks = computed(() => {
    const tasks = [...this.filteredTasks()];
    const sortBy = this._sortBy();

    return tasks.sort((a, b) => {
      if (sortBy === 'title') {
        return a.title.localeCompare(b.title);
      }
      if (sortBy === 'priority') {
        const priorityOrder = { low: 0, medium: 1, high: 2 };
        return priorityOrder[b.priority] - priorityOrder[a.priority];
      }
      return a.dueDate.getTime() - b.dueDate.getTime();
    });
  });

  // Computed: statistics
  readonly stats = computed(() => {
    const tasks = this._tasks();
    return {
      total: tasks.length,
      active: tasks.filter(t => !t.completed).length,
      completed: tasks.filter(t => t.completed).length,
      highPriority: tasks.filter(t => t.priority === 'high' && !t.completed).length
    };
  });

  // Computed: overdue tasks
  readonly overdueTasks = computed(() => {
    const now = new Date();
    return this._tasks().filter(t =>
      !t.completed && t.dueDate < now
    );
  });

  // Methods
  addTask(task: Task): void {
    this._tasks.update(tasks => [...tasks, task]);
  }

  updateTask(id: string, updates: Partial<Task>): void {
    this._tasks.update(tasks =>
      tasks.map(t => t.id === id ? { ...t, ...updates } : t)
    );
  }

  toggleTask(id: string): void {
    this._tasks.update(tasks =>
      tasks.map(t =>
        t.id === id ? { ...t, completed: !t.completed } : t
      )
    );
  }

  removeTask(id: string): void {
    this._tasks.update(tasks => tasks.filter(t => t.id !== id));
  }

  setFilter(filter: 'all' | 'active' | 'completed'): void {
    this._filter.set(filter);
  }

  setSortBy(sortBy: 'title' | 'priority' | 'dueDate'): void {
    this._sortBy.set(sortBy);
  }

  clearCompleted(): void {
    this._tasks.update(tasks => tasks.filter(t => !t.completed));
  }
}
```

### 3. Effects for Side Effects

```typescript
import { Injectable, signal, effect, Injector } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class ThemeStore {
  private readonly _theme = signal<'light' | 'dark'>('light');
  readonly theme = this._theme.asReadonly();

  constructor(private injector: Injector) {
    // Load from localStorage on init
    const saved = localStorage.getItem('theme');
    if (saved === 'light' || saved === 'dark') {
      this._theme.set(saved);
    }

    // Effect: save to localStorage when theme changes
    effect(() => {
      const theme = this._theme();
      localStorage.setItem('theme', theme);
      console.log('Theme saved:', theme);
    });

    // Effect: apply theme to document
    effect(() => {
      const theme = this._theme();
      document.body.setAttribute('data-theme', theme);
    });
  }

  toggleTheme(): void {
    this._theme.update(current => current === 'light' ? 'dark' : 'light');
  }

  setTheme(theme: 'light' | 'dark'): void {
    this._theme.set(theme);
  }
}

// Manual effect cleanup
@Injectable()
export class ComponentLevelStore {
  private readonly _value = signal(0);
  readonly value = this._value.asReadonly();

  constructor(private injector: Injector) {}

  startEffect(): EffectCleanupFn {
    // Effect with manual control
    return effect(() => {
      console.log('Value:', this._value());
    }, {
      injector: this.injector,
      manualCleanup: true
    });
  }

  // Usage:
  // const cleanup = this.startEffect();
  // cleanup(); // Stop the effect
}
```

### 4. Mutations and Updates

```typescript
import { Injectable, signal } from '@angular/core';

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

@Injectable({
  providedIn: 'root'
})
export class CartStore {
  private readonly _items = signal<CartItem[]>([]);
  readonly items = this._items.asReadonly();

  // Using set() - replaces entire value
  setItems(items: CartItem[]): void {
    this._items.set(items);
  }

  // Using update() - transforms current value
  addItem(item: Omit<CartItem, 'quantity'>): void {
    this._items.update(items => {
      const existing = items.find(i => i.productId === item.productId);
      
      if (existing) {
        return items.map(i =>
          i.productId === item.productId
            ? { ...i, quantity: i.quantity + 1 }
            : i
        );
      }
      
      return [...items, { ...item, quantity: 1 }];
    });
  }

  updateQuantity(productId: string, quantity: number): void {
    this._items.update(items => {
      if (quantity <= 0) {
        return items.filter(i => i.productId !== productId);
      }
      return items.map(i =>
        i.productId === productId ? { ...i, quantity } : i
      );
    });
  }

  removeItem(productId: string): void {
    this._items.update(items =>
      items.filter(i => i.productId !== productId)
    );
  }

  incrementQuantity(productId: string): void {
    this._items.update(items =>
      items.map(i =>
        i.productId === productId
          ? { ...i, quantity: i.quantity + 1 }
          : i
      )
    );
  }

  decrementQuantity(productId: string): void {
    this._items.update(items =>
      items.map(i =>
        i.productId === productId && i.quantity > 1
          ? { ...i, quantity: i.quantity - 1 }
          : i
      ).filter(i => i.quantity > 0)
    );
  }

  clear(): void {
    this._items.set([]);
  }
}
```

### 5. Nested State Management

```typescript
import { Injectable, signal, computed } from '@angular/core';

interface Address {
  street: string;
  city: string;
  state: string;
  zip: string;
}

interface UserProfile {
  name: string;
  email: string;
  address: Address;
  preferences: {
    notifications: boolean;
    theme: 'light' | 'dark';
  };
}

@Injectable({
  providedIn: 'root'
})
export class ProfileStore {
  private readonly _profile = signal<UserProfile>({
    name: '',
    email: '',
    address: {
      street: '',
      city: '',
      state: '',
      zip: ''
    },
    preferences: {
      notifications: true,
      theme: 'light'
    }
  });

  readonly profile = this._profile.asReadonly();

  // Computed for nested values
  readonly name = computed(() => this._profile().name);
  readonly email = computed(() => this._profile().email);
  readonly address = computed(() => this._profile().address);
  readonly preferences = computed(() => this._profile().preferences);
  readonly theme = computed(() => this._profile().preferences.theme);

  // Update top-level fields
  updateName(name: string): void {
    this._profile.update(profile => ({ ...profile, name }));
  }

  updateEmail(email: string): void {
    this._profile.update(profile => ({ ...profile, email }));
  }

  // Update nested address
  updateAddress(updates: Partial<Address>): void {
    this._profile.update(profile => ({
      ...profile,
      address: { ...profile.address, ...updates }
    }));
  }

  updateStreet(street: string): void {
    this._profile.update(profile => ({
      ...profile,
      address: { ...profile.address, street }
    }));
  }

  // Update nested preferences
  updatePreferences(updates: Partial<UserProfile['preferences']>): void {
    this._profile.update(profile => ({
      ...profile,
      preferences: { ...profile.preferences, ...updates }
    }));
  }

  toggleNotifications(): void {
    this._profile.update(profile => ({
      ...profile,
      preferences: {
        ...profile.preferences,
        notifications: !profile.preferences.notifications
      }
    }));
  }

  toggleTheme(): void {
    this._profile.update(profile => ({
      ...profile,
      preferences: {
        ...profile.preferences,
        theme: profile.preferences.theme === 'light' ? 'dark' : 'light'
      }
    }));
  }

  setProfile(profile: UserProfile): void {
    this._profile.set(profile);
  }

  reset(): void {
    this._profile.set({
      name: '',
      email: '',
      address: {
        street: '',
        city: '',
        state: '',
        zip: ''
      },
      preferences: {
        notifications: true,
        theme: 'light'
      }
    });
  }
}
```

## Component Integration

### Using Signals in Components

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-task-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="task-manager">
      <div class="controls">
        <button
          *ngFor="let f of ['all', 'active', 'completed']"
          [class.active]="filter() === f"
          (click)="taskStore.setFilter(f)"
        >
          {{ f }}
        </button>
      </div>

      <div class="sort">
        <select
          [value]="sortBy()"
          (change)="taskStore.setSortBy($any($event.target).value)"
        >
          <option value="title">Title</option>
          <option value="priority">Priority</option>
          <option value="dueDate">Due Date</option>
        </select>
      </div>

      <div class="stats">
        <p>Total: {{ stats().total }}</p>
        <p>Active: {{ stats().active }}</p>
        <p>Completed: {{ stats().completed }}</p>
        <p>High Priority: {{ stats().highPriority }}</p>
      </div>

      <div class="overdue" *ngIf="overdueTasks().length > 0">
        <h3>Overdue Tasks: {{ overdueTasks().length }}</h3>
      </div>

      <div class="task-list">
        <div
          *ngFor="let task of sortedTasks()"
          class="task-item"
          [class.completed]="task.completed"
        >
          <input
            type="checkbox"
            [checked]="task.completed"
            (change)="taskStore.toggleTask(task.id)"
          />
          <span class="title">{{ task.title }}</span>
          <span class="priority" [attr.data-priority]="task.priority">
            {{ task.priority }}
          </span>
          <span class="due-date">{{ task.dueDate | date }}</span>
          <button (click)="taskStore.removeTask(task.id)">Delete</button>
        </div>
      </div>

      <button
        *ngIf="stats().completed > 0"
        (click)="taskStore.clearCompleted()"
      >
        Clear Completed
      </button>
    </div>
  `
})
export class TaskListComponent {
  // Direct access to store signals - no subscriptions needed!
  sortedTasks = this.taskStore.sortedTasks;
  stats = this.taskStore.stats;
  overdueTasks = this.taskStore.overdueTasks;
  filter = this.taskStore.filter;
  sortBy = this.taskStore.sortBy;

  constructor(public taskStore: TaskStore) {}
}
```

### Signals with Forms

```typescript
import { Component, signal, computed } from '@angular/core';
import { FormsModule } from '@angular/forms';

interface FormErrors {
  name?: string;
  email?: string;
  password?: string;
}

@Component({
  selector: 'app-signup-form',
  standalone: true,
  imports: [FormsModule],
  template: `
    <form (ngSubmit)="handleSubmit()">
      <div>
        <input
          type="text"
          [value]="name()"
          (input)="setName($any($event.target).value)"
          placeholder="Name"
        />
        <span class="error" *ngIf="errors().name">
          {{ errors().name }}
        </span>
      </div>

      <div>
        <input
          type="email"
          [value]="email()"
          (input)="setEmail($any($event.target).value)"
          placeholder="Email"
        />
        <span class="error" *ngIf="errors().email">
          {{ errors().email }}
        </span>
      </div>

      <div>
        <input
          type="password"
          [value]="password()"
          (input)="setPassword($any($event.target).value)"
          placeholder="Password"
        />
        <span class="error" *ngIf="errors().password">
          {{ errors().password }}
        </span>
      </div>

      <button
        type="submit"
        [disabled]="!isValid() || submitting()"
      >
        {{ submitting() ? 'Submitting...' : 'Sign Up' }}
      </button>
    </form>
  `
})
export class SignupFormComponent {
  // Form fields
  name = signal('');
  email = signal('');
  password = signal('');
  submitting = signal(false);

  // Computed validation
  errors = computed<FormErrors>(() => {
    const errors: FormErrors = {};

    if (this.name().length < 2) {
      errors.name = 'Name must be at least 2 characters';
    }

    if (!this.email().includes('@')) {
      errors.email = 'Invalid email address';
    }

    if (this.password().length < 8) {
      errors.password = 'Password must be at least 8 characters';
    }

    return errors;
  });

  isValid = computed(() => {
    return Object.keys(this.errors()).length === 0 &&
           this.name() !== '' &&
           this.email() !== '' &&
           this.password() !== '';
  });

  setName(value: string): void {
    this.name.set(value);
  }

  setEmail(value: string): void {
    this.email.set(value);
  }

  setPassword(value: string): void {
    this.password.set(value);
  }

  handleSubmit(): void {
    if (!this.isValid()) return;

    this.submitting.set(true);

    // Simulate API call
    setTimeout(() => {
      console.log('Form submitted:', {
        name: this.name(),
        email: this.email(),
        password: this.password()
      });
      this.submitting.set(false);
      this.resetForm();
    }, 2000);
  }

  resetForm(): void {
    this.name.set('');
    this.email.set('');
    this.password.set('');
  }
}
```

## Advanced Patterns

### Async Operations with Signals

```typescript
import { Injectable, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { lastValueFrom } from 'rxjs';

interface ApiState<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

@Injectable({
  providedIn: 'root'
})
export class DataStore {
  private readonly _users = signal<ApiState<User[]>>({
    data: null,
    loading: false,
    error: null
  });

  readonly users = computed(() => this._users().data);
  readonly loading = computed(() => this._users().loading);
  readonly error = computed(() => this._users().error);
  readonly hasData = computed(() => this._users().data !== null);

  constructor(private http: HttpClient) {}

  async loadUsers(): Promise<void> {
    this._users.update(state => ({ ...state, loading: true, error: null }));

    try {
      const data = await lastValueFrom(
        this.http.get<User[]>('/api/users')
      );
      
      this._users.set({ data, loading: false, error: null });
    } catch (error) {
      this._users.set({
        data: null,
        loading: false,
        error: error instanceof Error ? error.message : 'Unknown error'
      });
    }
  }

  async createUser(user: Omit<User, 'id'>): Promise<void> {
    this._users.update(state => ({ ...state, loading: true }));

    try {
      const newUser = await lastValueFrom(
        this.http.post<User>('/api/users', user)
      );

      this._users.update(state => ({
        data: state.data ? [...state.data, newUser] : [newUser],
        loading: false,
        error: null
      }));
    } catch (error) {
      this._users.update(state => ({
        ...state,
        loading: false,
        error: error instanceof Error ? error.message : 'Unknown error'
      }));
    }
  }

  clearError(): void {
    this._users.update(state => ({ ...state, error: null }));
  }
}
```

### Signal-based Pagination

```typescript
@Injectable({
  providedIn: 'root'
})
export class PaginationStore {
  private readonly _items = signal<string[]>([]);
  private readonly _page = signal(1);
  private readonly _pageSize = signal(10);

  readonly items = this._items.asReadonly();
  readonly page = this._page.asReadonly();
  readonly pageSize = this._pageSize.asReadonly();

  // Computed pagination values
  readonly totalItems = computed(() => this._items().length);
  
  readonly totalPages = computed(() =>
    Math.ceil(this.totalItems() / this._pageSize())
  );

  readonly startIndex = computed(() =>
    (this._page() - 1) * this._pageSize()
  );

  readonly endIndex = computed(() =>
    Math.min(this.startIndex() + this._pageSize(), this.totalItems())
  );

  readonly paginatedItems = computed(() => {
    const items = this._items();
    return items.slice(this.startIndex(), this.endIndex());
  });

  readonly hasPrevious = computed(() => this._page() > 1);
  
  readonly hasNext = computed(() =>
    this._page() < this.totalPages()
  );

  readonly pageInfo = computed(() => ({
    page: this._page(),
    pageSize: this._pageSize(),
    totalItems: this.totalItems(),
    totalPages: this.totalPages(),
    startIndex: this.startIndex(),
    endIndex: this.endIndex(),
    hasPrevious: this.hasPrevious(),
    hasNext: this.hasNext()
  }));

  setItems(items: string[]): void {
    this._items.set(items);
    this._page.set(1); // Reset to first page
  }

  setPage(page: number): void {
    const totalPages = this.totalPages();
    if (page >= 1 && page <= totalPages) {
      this._page.set(page);
    }
  }

  setPageSize(pageSize: number): void {
    this._pageSize.set(pageSize);
    this._page.set(1); // Reset to first page
  }

  nextPage(): void {
    if (this.hasNext()) {
      this._page.update(p => p + 1);
    }
  }

  previousPage(): void {
    if (this.hasPrevious()) {
      this._page.update(p => p - 1);
    }
  }

  goToFirstPage(): void {
    this._page.set(1);
  }

  goToLastPage(): void {
    this._page.set(this.totalPages());
  }
}
```

### Signal-based Undo/Redo

```typescript
@Injectable({
  providedIn: 'root'
})
export class UndoRedoStore {
  private readonly _history = signal<{
    past: string[];
    present: string;
    future: string[];
  }>({
    past: [],
    present: '',
    future: []
  });

  readonly present = computed(() => this._history().present);
  readonly canUndo = computed(() => this._history().past.length > 0);
  readonly canRedo = computed(() => this._history().future.length > 0);

  setValue(value: string): void {
    this._history.update(({ past, present }) => ({
      past: [...past, present],
      present: value,
      future: [] // Clear future on new change
    }));
  }

  undo(): void {
    if (!this.canUndo()) return;

    this._history.update(({ past, present, future }) => {
      const previous = past[past.length - 1];
      const newPast = past.slice(0, -1);

      return {
        past: newPast,
        present: previous,
        future: [present, ...future]
      };
    });
  }

  redo(): void {
    if (!this.canRedo()) return;

    this._history.update(({ past, present, future }) => {
      const next = future[0];
      const newFuture = future.slice(1);

      return {
        past: [...past, present],
        present: next,
        future: newFuture
      };
    });
  }

  reset(): void {
    this._history.set({
      past: [],
      present: '',
      future: []
    });
  }
}
```

## Common Mistakes

1. **Calling signal as function in updates**: Use `signal()` to read, not `signal` itself.

2. **Not using computed for derived values**: Computing in templates is inefficient.

3. **Mutating signal values**: Always use `set()` or `update()` with new objects/arrays.

4. **Overusing effects**: Effects should be for side effects, not derived state (use computed instead).

5. **Not making signals readonly**: Expose readonly versions to prevent external mutations.

## Best Practices

1. **Keep signals private**: Expose computed or readonly versions publicly
2. **Use computed for derivations**: Never manually track dependencies
3. **Immutable updates**: Always create new objects/arrays in update()
4. **Effect cleanup**: Use manualCleanup for long-running effects
5. **Granular signals**: Prefer multiple focused signals over one large signal
6. **Async handling**: Use async/await with lastValueFrom for HTTP calls
7. **Type safety**: Always define interfaces for signal values
8. **Testing**: Signals are synchronous and easy to test

## When to Use Signals Store

**Use Signals Store when:**

- Angular 16+ applications
- Need fine-grained reactivity
- Want synchronous state updates
- Building component-level state
- Performance is critical
- Want to eliminate Zone.js

**Consider alternatives when:**

- Need time-travel debugging (use NgRx)
- Complex async workflows (use Observables)
- Team unfamiliar with Signals
- Angular version < 16

## Interview Questions

1. **Q: How do Signals differ from Observables?**
   A: Signals are synchronous, pull-based, and always have a current value. Observables are asynchronous, push-based, and may not have emitted yet. Signals enable fine-grained reactivity without Zone.js.

2. **Q: When should you use computed vs effects?**
   A: Use computed for derived values that depend on other signals (pure functions). Use effects for side effects like DOM manipulation, logging, or synchronization with external systems.

3. **Q: How does signal equality checking work?**
   A: Signals use reference equality by default. Use `equal` option to provide custom equality function. Computed signals cache results and only recompute when dependencies change.

4. **Q: Can signals replace RxJS entirely?**
   A: No. Signals excel at synchronous state, but RxJS is better for complex async operations, timing, combining multiple streams, and HTTP. They complement each other.

5. **Q: How do you make signals readonly?**
   A: Use `signal.asReadonly()` or expose only computed signals that wrap the private writable signal. This prevents external code from calling `set()` or `update()`.

6. **Q: What's the performance benefit of signals?**
   A: Signals enable fine-grained reactivity where only components consuming changed signals update, without Zone.js overhead. This is more efficient than zone-based change detection.

## Key Takeaways

1. Signals provide synchronous, fine-grained reactivity in Angular 16+
2. Use signal() for state, computed() for derived values, effect() for side effects
3. Signals are pull-based and always have a current value
4. Computed signals automatically track dependencies and memoize results
5. Effects run when their signal dependencies change
6. Immutable updates with set() and update() are required
7. Expose readonly signals publicly, keep writable signals private
8. Signals enable zoneless Angular applications
9. Better performance than Observable-based state for synchronous operations
10. Signals and Observables complement each other in modern Angular

## Resources

- [Angular Signals Documentation](https://angular.io/guide/signals)
- [NgRx SignalStore](https://ngrx.io/guide/signals)
- [Signals Deep Dive](https://blog.angular.io/angular-v16-is-here-4d7a28ec680d)
- [Zoneless Angular with Signals](https://angular.io/guide/zoneless)
- [Signals vs Observables](https://blog.angular-university.io/angular-signals/)
