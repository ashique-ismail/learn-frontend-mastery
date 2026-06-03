# toSignal and toObservable - RxJS Interop in Angular

## Table of Contents
- [Introduction](#introduction)
- [toSignal() Function](#tosignal-function)
- [toObservable() Function](#toobservable-function)
- [Interop Patterns](#interop-patterns)
- [Migration Strategies](#migration-strategies)
- [Performance Considerations](#performance-considerations)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Angular provides seamless interoperability between Signals and RxJS Observables through `toSignal()` and `toObservable()`. These utilities enable gradual migration from RxJS to Signals while maintaining backward compatibility with existing RxJS-based code.

**Key Use Cases:**
- Wrapping HTTP calls in signals
- Converting existing Observable-based services to signals
- Mixing reactive paradigms when needed
- Gradual migration from RxJS to Signals
- Integrating third-party Observable libraries

## toSignal() Function

### Basic Usage

```typescript
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-basic-tosignal',
  template: `
    <div>
      @if (users()) {
        <div *ngFor="let user of users()">
          {{ user.name }}
        </div>
      } @else {
        <p>Loading...</p>
      }
    </div>
  `
})
export class BasicToSignalComponent {
  private http = inject(HttpClient);
  
  // Convert Observable to Signal
  users = toSignal(
    this.http.get<User[]>('/api/users')
  );
}
```

### Initial Values

```typescript
import { Component } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-initial-value',
  template: `
    <div>
      <!-- Always has a value, never undefined -->
      <p>Count: {{ count() }}</p>
      <p>Timestamp: {{ timestamp() }}</p>
    </div>
  `
})
export class InitialValueComponent {
  // With initial value - signal is never undefined
  count = toSignal(interval(1000), { initialValue: 0 });
  
  // Without initial value - signal might be undefined initially
  timestamp = toSignal(interval(1000));
  
  // Type of count: Signal<number>
  // Type of timestamp: Signal<number | undefined>
}
```

### Required Values

```typescript
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';

interface Config {
  apiUrl: string;
  timeout: number;
}

@Component({
  selector: 'app-required-signal',
  template: `
    <div>
      <!-- Config is guaranteed to have a value -->
      <p>API URL: {{ config().apiUrl }}</p>
      <p>Timeout: {{ config().timeout }}ms</p>
    </div>
  `
})
export class RequiredSignalComponent {
  private http = inject(HttpClient);
  
  // Using requireSync - throws if Observable doesn't emit synchronously
  config = toSignal(
    this.http.get<Config>('/api/config'),
    { requireSync: true } // Must emit immediately
  );
  
  // Better: Provide initial value
  configWithDefault = toSignal(
    this.http.get<Config>('/api/config'),
    {
      initialValue: {
        apiUrl: 'https://default-api.com',
        timeout: 30000
      }
    }
  );
}
```

### Manual Subscription Management

```typescript
import { Component, inject, Injector, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-manual-subscription',
  template: `
    <div>
      <p>Counter: {{ counter() }}</p>
      <button (click)="start()">Start</button>
      <button (click)="stop()">Stop</button>
    </div>
  `
})
export class ManualSubscriptionComponent {
  private injector = inject(Injector);
  
  counter = signal<number | undefined>(undefined);
  private subscription: any;
  
  start(): void {
    if (this.subscription) return;
    
    // Create signal with manual injector
    this.counter = toSignal(interval(1000), {
      injector: this.injector,
      initialValue: 0
    });
  }
  
  stop(): void {
    // toSignal automatically unsubscribes when component destroys
    // For manual control, recreate the signal
    this.counter.set(undefined);
  }
}
```

### HTTP with toSignal

```typescript
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { switchMap, of } from 'rxjs';

interface Post {
  id: number;
  title: string;
  body: string;
}

@Component({
  selector: 'app-http-signal',
  template: `
    <div>
      <input 
        type="number" 
        [(ngModel)]="postIdValue"
        (ngModelChange)="postId.set(+postIdValue)">
      
      @if (post(); as p) {
        <article>
          <h2>{{ p.title }}</h2>
          <p>{{ p.body }}</p>
        </article>
      } @else {
        <p>Enter a post ID</p>
      }
    </div>
  `
})
export class HttpSignalComponent {
  private http = inject(HttpClient);
  
  postId = signal<number | null>(null);
  postIdValue: number | null = null;
  
  // Convert HTTP Observable to Signal
  post = toSignal(
    toObservable(this.postId).pipe(
      switchMap(id => 
        id ? this.http.get<Post>(`/api/posts/${id}`) : of(null)
      )
    )
  );
}
```

### Error Handling

```typescript
import { Component, inject, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { catchError, of, map } from 'rxjs';

interface ApiResponse<T> {
  data: T | null;
  error: Error | null;
  loading: boolean;
}

@Component({
  selector: 'app-error-handling',
  template: `
    <div>
      @if (users().loading) {
        <p>Loading users...</p>
      } @else if (users().error) {
        <div class="error">
          <p>Error: {{ users().error?.message }}</p>
          <button (click)="retry()">Retry</button>
        </div>
      } @else if (users().data) {
        <div *ngFor="let user of users().data">
          {{ user.name }}
        </div>
      }
    </div>
  `
})
export class ErrorHandlingComponent {
  private http = inject(HttpClient);
  private refreshTrigger = signal(0);
  
  users = toSignal(
    toObservable(this.refreshTrigger).pipe(
      switchMap(() =>
        this.http.get<User[]>('/api/users').pipe(
          map(data => ({ data, error: null, loading: false })),
          catchError(error => of({ data: null, error, loading: false }))
        )
      )
    ),
    { initialValue: { data: null, error: null, loading: true } as ApiResponse<User[]> }
  );
  
  retry(): void {
    this.refreshTrigger.update(n => n + 1);
  }
}
```

### Reactive Queries Pattern

```typescript
import { Component, inject, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { switchMap, debounceTime, distinctUntilChanged, map } from 'rxjs';

@Component({
  selector: 'app-reactive-query',
  template: `
    <div>
      <input 
        [(ngModel)]="searchTermValue"
        (ngModelChange)="searchTerm.set(searchTermValue)"
        placeholder="Search...">
      
      <select [(ngModel)]="categoryValue" (change)="category.set(categoryValue)">
        <option value="">All</option>
        <option value="tech">Tech</option>
        <option value="science">Science</option>
      </select>
      
      @if (results()?.length) {
        <div *ngFor="let result of results()">
          {{ result.title }}
        </div>
      } @else {
        <p>No results</p>
      }
    </div>
  `
})
export class ReactiveQueryComponent {
  private http = inject(HttpClient);
  
  searchTerm = signal('');
  category = signal('');
  searchTermValue = '';
  categoryValue = '';
  
  // Combine multiple signals into query parameters
  queryParams = computed(() => ({
    search: this.searchTerm(),
    category: this.category()
  }));
  
  // Convert to Observable and perform reactive query
  results = toSignal(
    toObservable(this.queryParams).pipe(
      debounceTime(300),
      distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b)),
      switchMap(params => {
        const searchParams: any = {};
        if (params.search) searchParams.q = params.search;
        if (params.category) searchParams.category = params.category;
        
        return this.http.get<any[]>('/api/search', { params: searchParams });
      })
    ),
    { initialValue: [] }
  );
}
```

## toObservable() Function

### Basic Usage

```typescript
import { Component, signal } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { debounceTime, map } from 'rxjs';

@Component({
  selector: 'app-basic-toobservable',
  template: `
    <div>
      <input [(ngModel)]="value" (ngModelChange)="searchTerm.set(value)">
      <p>Search term: {{ searchTerm() }}</p>
    </div>
  `
})
export class BasicToObservableComponent {
  searchTerm = signal('');
  value = '';
  
  // Convert signal to Observable
  searchTerm$ = toObservable(this.searchTerm);
  
  // Use RxJS operators
  debouncedSearch$ = this.searchTerm$.pipe(
    debounceTime(300),
    map(term => term.toLowerCase())
  );
  
  constructor() {
    // Subscribe to Observable
    this.debouncedSearch$.subscribe(term => {
      console.log('Debounced search:', term);
    });
  }
}
```

### Combining Signals into Observable

```typescript
import { Component, signal, computed } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { combineLatest, map } from 'rxjs';

@Component({
  selector: 'app-combine-signals',
  template: `
    <div>
      <input type="number" [(ngModel)]="widthValue" (ngModelChange)="width.set(+widthValue)">
      <input type="number" [(ngModel)]="heightValue" (ngModelChange)="height.set(+heightValue)">
      <p>Area: {{ area() }}</p>
    </div>
  `
})
export class CombineSignalsComponent {
  width = signal(10);
  height = signal(10);
  widthValue = 10;
  heightValue = 10;
  
  // Convert signals to Observables
  width$ = toObservable(this.width);
  height$ = toObservable(this.height);
  
  // Combine using RxJS
  area$ = combineLatest([this.width$, this.height$]).pipe(
    map(([w, h]) => w * h)
  );
  
  // Or use computed signal (better approach)
  area = computed(() => this.width() * this.height());
  
  constructor() {
    this.area$.subscribe(area => {
      console.log('Area changed:', area);
    });
  }
}
```

### Integration with RxJS Services

```typescript
import { Injectable, signal } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { Observable, switchMap } from 'rxjs';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  
  // Signal-based state
  selectedUserId = signal<number | null>(null);
  
  // Convert to Observable for RxJS operations
  selectedUserId$ = toObservable(this.selectedUserId);
  
  // Reactive user loading
  selectedUser$: Observable<User | null> = this.selectedUserId$.pipe(
    switchMap(id => 
      id ? this.http.get<User>(`/api/users/${id}`) : of(null)
    )
  );
  
  selectUser(id: number): void {
    this.selectedUserId.set(id);
  }
}

@Component({
  selector: 'app-user-details',
  template: `
    <div>
      <button (click)="selectUser(1)">User 1</button>
      <button (click)="selectUser(2)">User 2</button>
      
      @if (user$ | async; as user) {
        <div>
          <h3>{{ user.name }}</h3>
          <p>{{ user.email }}</p>
        </div>
      }
    </div>
  `
})
export class UserDetailsComponent {
  private userService = inject(UserService);
  
  user$ = this.userService.selectedUser$;
  
  selectUser(id: number): void {
    this.userService.selectUser(id);
  }
}
```

### Side Effects with toObservable

```typescript
import { Component, signal, effect } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { tap, distinctUntilChanged } from 'rxjs';

@Component({
  selector: 'app-side-effects',
  template: `
    <div>
      <button (click)="count.update(c => c + 1)">Increment: {{ count() }}</button>
    </div>
  `
})
export class SideEffectsComponent {
  count = signal(0);
  
  // Approach 1: Using effect (preferred for signals)
  constructor() {
    effect(() => {
      const current = this.count();
      console.log('Count changed (effect):', current);
      localStorage.setItem('count', current.toString());
    });
  }
  
  // Approach 2: Using toObservable (useful for RxJS operators)
  count$ = toObservable(this.count).pipe(
    distinctUntilChanged(),
    tap(count => {
      console.log('Count changed (observable):', count);
      // Trigger analytics
      if (count % 10 === 0) {
        console.log('Milestone reached!');
      }
    })
  );
}
```

## Interop Patterns

### Bi-directional Conversion

```typescript
import { Component, signal, inject } from '@angular/core';
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { switchMap, debounceTime } from 'rxjs';

@Component({
  selector: 'app-bidirectional',
  template: `
    <div>
      <input 
        [(ngModel)]="searchTermValue"
        (ngModelChange)="searchTerm.set(searchTermValue)"
        placeholder="Search users...">
      
      @if (users()) {
        <div *ngFor="let user of users()">
          {{ user.name }}
        </div>
      }
    </div>
  `
})
export class BidirectionalComponent {
  private http = inject(HttpClient);
  
  // Signal → Observable → Signal cycle
  searchTerm = signal('');
  searchTermValue = '';
  
  // Convert signal to Observable for RxJS operators
  searchTerm$ = toObservable(this.searchTerm).pipe(
    debounceTime(300)
  );
  
  // Use Observable for HTTP, convert back to Signal
  users = toSignal(
    this.searchTerm$.pipe(
      switchMap(term => 
        term ? this.http.get<User[]>(`/api/users?search=${term}`) : of([])
      )
    ),
    { initialValue: [] }
  );
}
```

### Mixing Paradigms

```typescript
import { Component, signal, computed, inject } from '@angular/core';
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { FormControl } from '@angular/forms';
import { debounceTime, startWith } from 'rxjs';

@Component({
  selector: 'app-mixed-paradigms',
  template: `
    <div>
      <!-- Angular Forms (Observable-based) -->
      <input [formControl]="emailControl">
      
      <!-- Signal-based display -->
      <p>Email: {{ email() }}</p>
      <p>Valid: {{ isValid() }}</p>
      
      <!-- Computed from mixed sources -->
      <p>Display name: {{ displayName() }}</p>
    </div>
  `
})
export class MixedParadigmsComponent {
  // Observable-based form control
  emailControl = new FormControl('');
  
  // Convert Observable to Signal
  email = toSignal(
    this.emailControl.valueChanges.pipe(
      startWith(''),
      debounceTime(300)
    ),
    { initialValue: '' }
  );
  
  // Signal-based validation
  isValid = computed(() => {
    const email = this.email();
    return email.includes('@') && email.length > 5;
  });
  
  // Pure signal
  firstName = signal('John');
  
  // Computed from both paradigms
  displayName = computed(() => {
    const email = this.email();
    const name = this.firstName();
    return email ? `${name} (${email})` : name;
  });
}
```

### Service Layer Integration

```typescript
import { Injectable, signal, inject } from '@angular/core';
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject, combineLatest, switchMap, map } from 'rxjs';

interface FilterState {
  search: string;
  category: string;
  sortBy: string;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  
  // Signal-based state
  private searchTerm = signal('');
  private category = signal('');
  
  // Observable-based state (legacy code)
  private sortBy$ = new BehaviorSubject<string>('name');
  
  // Convert signals to Observables
  private searchTerm$ = toObservable(this.searchTerm);
  private category$ = toObservable(this.category);
  
  // Combine all sources
  private filterState$ = combineLatest([
    this.searchTerm$,
    this.category$,
    this.sortBy$
  ]).pipe(
    map(([search, category, sortBy]) => ({ search, category, sortBy }))
  );
  
  // Reactive data loading
  private products$ = this.filterState$.pipe(
    switchMap(filters => this.loadProducts(filters))
  );
  
  // Convert back to signal for components
  products = toSignal(this.products$, { initialValue: [] });
  
  // Public API using signals
  setSearch(term: string): void {
    this.searchTerm.set(term);
  }
  
  setCategory(cat: string): void {
    this.category.set(cat);
  }
  
  // Public API using Observables (legacy)
  setSortBy(sort: string): void {
    this.sortBy$.next(sort);
  }
  
  private loadProducts(filters: FilterState): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products', { params: filters as any });
  }
}

@Component({
  selector: 'app-product-list',
  template: `
    <div>
      <input (input)="onSearch($any($event.target).value)">
      <select (change)="onCategoryChange($any($event.target).value)">
        <option value="">All</option>
        <option value="tech">Tech</option>
      </select>
      
      <div *ngFor="let product of productService.products()">
        {{ product.name }}
      </div>
    </div>
  `
})
export class ProductListComponent {
  productService = inject(ProductService);
  
  onSearch(term: string): void {
    this.productService.setSearch(term);
  }
  
  onCategoryChange(category: string): void {
    this.productService.setCategory(category);
  }
}
```

## Migration Strategies

### Gradual Migration Approach

```typescript
// Phase 1: Keep Observable API, use signals internally
@Injectable({ providedIn: 'root' })
export class DataServicePhase1 {
  private dataSignal = signal<Data[]>([]);
  
  // Keep Observable interface for backward compatibility
  data$ = toObservable(this.dataSignal);
  
  updateData(data: Data[]): void {
    this.dataSignal.set(data);
  }
}

// Phase 2: Expose both APIs
@Injectable({ providedIn: 'root' })
export class DataServicePhase2 {
  private dataSignal = signal<Data[]>([]);
  
  // New signal-based API
  data = this.dataSignal.asReadonly();
  
  // Deprecated Observable API
  /** @deprecated Use data signal instead */
  data$ = toObservable(this.dataSignal);
  
  updateData(data: Data[]): void {
    this.dataSignal.set(data);
  }
}

// Phase 3: Signal-only API
@Injectable({ providedIn: 'root' })
export class DataServicePhase3 {
  private dataSignal = signal<Data[]>([]);
  
  // Signal-only API
  readonly data = this.dataSignal.asReadonly();
  
  updateData(data: Data[]): void {
    this.dataSignal.set(data);
  }
}
```

### Component Migration Pattern

```typescript
// Before: Observable-based component
@Component({
  selector: 'app-old',
  template: `
    <div *ngFor="let item of items$ | async">
      {{ item.name }}
    </div>
  `
})
export class OldComponent {
  private dataService = inject(DataService);
  
  items$ = this.dataService.getItems();
}

// After: Signal-based component
@Component({
  selector: 'app-new',
  template: `
    <div *ngFor="let item of items()">
      {{ item.name }}
    </div>
  `
})
export class NewComponent {
  private dataService = inject(DataService);
  
  // Convert Observable to Signal
  items = toSignal(this.dataService.getItems(), { initialValue: [] });
}

// Better: Service provides signals directly
@Component({
  selector: 'app-best',
  template: `
    <div *ngFor="let item of dataService.items()">
      {{ item.name }}
    </div>
  `
})
export class BestComponent {
  dataService = inject(DataService); // Service already provides signals
}
```

## Performance Considerations

### Subscription Management

```typescript
import { Component, signal, OnDestroy } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-subscription-management',
  template: '<p>{{ count() }}</p>'
})
export class SubscriptionManagementComponent implements OnDestroy {
  count = signal(0);
  
  // toObservable creates a subscription
  count$ = toObservable(this.count);
  
  private subscription: Subscription;
  
  constructor() {
    // Manual subscription needs cleanup
    this.subscription = this.count$.subscribe(c => {
      console.log('Count:', c);
    });
  }
  
  ngOnDestroy(): void {
    // Clean up manual subscriptions
    this.subscription.unsubscribe();
  }
}

// Better: Use effect for side effects
@Component({
  selector: 'app-better-approach',
  template: '<p>{{ count() }}</p>'
})
export class BetterApproachComponent {
  count = signal(0);
  
  constructor() {
    // effect automatically cleans up
    effect(() => {
      console.log('Count:', this.count());
    });
  }
}
```

### Change Detection Optimization

```typescript
import { Component, signal, ChangeDetectionStrategy } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-optimized',
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <p>Timestamp: {{ timestamp() }}</p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedComponent {
  // Signals work perfectly with OnPush
  count = signal(0);
  
  // toSignal also triggers change detection appropriately
  timestamp = toSignal(interval(1000), { initialValue: 0 });
  
  increment(): void {
    this.count.update(c => c + 1);
    // Only this component updates, not the entire tree
  }
}
```

### Memory Considerations

```typescript
import { Component, signal, inject, DestroyRef } from '@angular/core';
import { toObservable, toSignal, takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-memory-safe',
  template: '<p>{{ data() }}</p>'
})
export class MemorySafeComponent {
  private destroyRef = inject(DestroyRef);
  
  trigger = signal(0);
  
  // toSignal automatically cleans up on component destroy
  data = toSignal(
    toObservable(this.trigger).pipe(
      switchMap(() => interval(1000))
    ),
    { initialValue: 0 }
  );
  
  // Manual Observable subscription with cleanup
  manualSubscription$ = toObservable(this.trigger).pipe(
    takeUntilDestroyed(this.destroyRef)
  ).subscribe();
}
```

## Advanced Patterns

### Custom Reactive Store

```typescript
import { Injectable, signal, computed } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged } from 'rxjs';

interface StoreState {
  loading: boolean;
  data: any[];
  error: Error | null;
}

@Injectable({ providedIn: 'root' })
export class ReactiveStore {
  // Private state
  private state = signal<StoreState>({
    loading: false,
    data: [],
    error: null
  });
  
  // Public selectors (signals)
  loading = computed(() => this.state().loading);
  data = computed(() => this.state().data);
  error = computed(() => this.state().error);
  
  // Observable interface for RxJS operators
  state$ = toObservable(this.state);
  
  // Debounced state changes
  debouncedState$ = this.state$.pipe(
    debounceTime(300),
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );
  
  // Actions
  setLoading(loading: boolean): void {
    this.state.update(s => ({ ...s, loading }));
  }
  
  setData(data: any[]): void {
    this.state.update(s => ({ ...s, data, loading: false, error: null }));
  }
  
  setError(error: Error): void {
    this.state.update(s => ({ ...s, error, loading: false }));
  }
}
```

### WebSocket Integration

```typescript
import { Injectable, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { Observable, Subject } from 'rxjs';
import { retry, tap } from 'rxjs/operators';

interface Message {
  type: string;
  payload: any;
}

@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private ws$: WebSocketSubject<Message>;
  private messagesSubject = new Subject<Message>();
  
  // Convert Observable to Signal
  messages = toSignal(this.messagesSubject.asObservable(), { initialValue: null });
  
  connectionStatus = signal<'connected' | 'disconnected' | 'connecting'>('disconnected');
  
  connect(url: string): void {
    this.connectionStatus.set('connecting');
    
    this.ws$ = webSocket({
      url,
      openObserver: {
        next: () => this.connectionStatus.set('connected')
      },
      closeObserver: {
        next: () => this.connectionStatus.set('disconnected')
      }
    });
    
    this.ws$.pipe(
      retry({ delay: 3000 }),
      tap(message => this.messagesSubject.next(message))
    ).subscribe();
  }
  
  send(message: Message): void {
    this.ws$.next(message);
  }
  
  disconnect(): void {
    this.ws$.complete();
    this.connectionStatus.set('disconnected');
  }
}

@Component({
  selector: 'app-websocket-client',
  template: `
    <div>
      <p>Status: {{ ws.connectionStatus() }}</p>
      
      @if (ws.messages(); as msg) {
        <div>{{ msg.type }}: {{ msg.payload | json }}</div>
      }
      
      <button (click)="send()">Send Message</button>
    </div>
  `
})
export class WebSocketClientComponent {
  ws = inject(WebSocketService);
  
  constructor() {
    this.ws.connect('wss://api.example.com/ws');
  }
  
  send(): void {
    this.ws.send({ type: 'ping', payload: {} });
  }
}
```

## Common Mistakes

### Mistake 1: Not Providing Initial Value

```typescript
// BAD: Signal might be undefined
const data = toSignal(http.get('/api/data'));
console.log(data().length); // Runtime error if undefined!

// GOOD: Provide initial value
const data = toSignal(http.get('/api/data'), { initialValue: [] });
console.log(data().length); // Always safe
```

### Mistake 2: Unnecessary Conversions

```typescript
// BAD: Converting back and forth unnecessarily
const count = signal(0);
const count$ = toObservable(count);
const countSignal = toSignal(count$, { initialValue: 0 });

// GOOD: Use signals directly
const count = signal(0);
const doubled = computed(() => count() * 2);
```

### Mistake 3: Memory Leaks with Manual Subscriptions

```typescript
// BAD: No cleanup
@Component({...})
export class BadComponent {
  count = signal(0);
  
  constructor() {
    toObservable(this.count).subscribe(c => {
      console.log(c); // Subscription never cleaned up!
    });
  }
}

// GOOD: Use effect or takeUntilDestroyed
@Component({...})
export class GoodComponent {
  count = signal(0);
  
  constructor() {
    effect(() => {
      console.log(this.count()); // Auto cleanup
    });
  }
}
```

### Mistake 4: Misusing requireSync

```typescript
// BAD: Using requireSync with async operations
const data = toSignal(
  http.get('/api/data'),
  { requireSync: true } // Will throw! HTTP is async
);

// GOOD: Use initial value instead
const data = toSignal(
  http.get('/api/data'),
  { initialValue: [] }
);
```

### Mistake 5: Ignoring Type Safety

```typescript
// BAD: Losing type information
const data = toSignal(http.get<User[]>('/api/users'));
// Type: Signal<User[] | undefined>

// GOOD: Maintain type safety with initialValue
const data = toSignal(
  http.get<User[]>('/api/users'),
  { initialValue: [] as User[] }
);
// Type: Signal<User[]>
```

## Best Practices

### 1. Prefer Signals Over Observables for State

```typescript
// Use signals for synchronous state
const count = signal(0);
const items = signal<Item[]>([]);

// Use Observables only for async operations
const data$ = http.get('/api/data');
```

### 2. Convert Observables at the Edge

```typescript
// Convert Observables to Signals at component boundary
@Component({...})
export class MyComponent {
  private http = inject(HttpClient);
  
  // Convert once at the edge
  users = toSignal(
    this.http.get<User[]>('/api/users'),
    { initialValue: [] }
  );
  
  // Then use signals throughout component
  activeUsers = computed(() => 
    this.users().filter(u => u.active)
  );
}
```

### 3. Use Effect for Signal-Based Side Effects

```typescript
// Prefer effect over toObservable for side effects
const count = signal(0);

// GOOD: Using effect
effect(() => {
  console.log('Count:', count());
  localStorage.setItem('count', count().toString());
});

// AVOID: Using toObservable for simple side effects
toObservable(count).subscribe(c => {
  console.log('Count:', c);
});
```

### 4. Provide Initial Values

```typescript
// Always provide initialValue for better type safety
const users = toSignal(http.get<User[]>('/api/users'), {
  initialValue: []
});

// Type is Signal<User[]>, not Signal<User[] | undefined>
```

### 5. Document Conversion Points

```typescript
@Injectable({ providedIn: 'root' })
export class DataService {
  private http = inject(HttpClient);
  
  /**
   * Loads users from API
   * @returns Signal with user array (empty during loading)
   */
  loadUsers(): Signal<User[]> {
    // Conversion point documented
    return toSignal(
      this.http.get<User[]>('/api/users'),
      { initialValue: [] }
    );
  }
}
```

## Interview Questions

### Q1: What is the difference between toSignal() and toObservable()?

**Answer:** `toSignal()` converts an Observable to a Signal, enabling use in reactive contexts with automatic subscription management. `toObservable()` converts a Signal to an Observable, useful for applying RxJS operators or integrating with Observable-based APIs. Use toSignal for consuming async data in components, toObservable for complex RxJS transformations.

### Q2: Why should you provide an initialValue to toSignal()?

**Answer:** Providing initialValue ensures the signal always has a value and improves type safety (Signal<T> instead of Signal<T | undefined>). It prevents undefined errors in templates and makes the signal immediately usable without null checks.

### Q3: When should you use toObservable()?

**Answer:** Use toObservable() when you need RxJS operators (debounceTime, switchMap, etc.), when integrating with Observable-based libraries, or when you need to combine multiple signals using RxJS operators. For simple transformations, prefer computed() signals.

### Q4: How does toSignal() handle subscriptions?

**Answer:** toSignal() automatically subscribes to the Observable and cleans up the subscription when the injection context (component/service) is destroyed. You don't need to manually unsubscribe, making it safer than manual subscriptions.

### Q5: Can you mix Signals and Observables in the same component?

**Answer:** Yes, and it's common during migration. Use toSignal() to convert Observables to Signals for reactive state, and toObservable() to convert Signals to Observables when you need RxJS operators. This allows gradual migration from RxJS to Signals.

## Key Takeaways

1. toSignal() converts Observables to Signals with auto-cleanup
2. toObservable() converts Signals to Observables for RxJS operators
3. Always provide initialValue to toSignal() for type safety
4. toSignal() automatically manages subscriptions
5. Use Signals for state, Observables for async operations
6. Prefer effect() over toObservable() for simple side effects
7. Convert at component boundaries for cleaner code
8. Enables gradual migration from RxJS to Signals
9. Both functions work with Angular's injection system
10. Essential for interop between reactive paradigms

## Resources

- [Angular RxJS Interop Guide](https://angular.dev/guide/signals/rxjs-interop)
- [toSignal API Documentation](https://angular.dev/api/core/rxjs-interop/toSignal)
- [toObservable API Documentation](https://angular.dev/api/core/rxjs-interop/toObservable)
- [Signals and RxJS Integration](https://blog.angular.io/signals-rxjs-integration)
- [Migration Guide](https://angular.dev/guide/signals/rxjs-interop#migration-from-observables)
