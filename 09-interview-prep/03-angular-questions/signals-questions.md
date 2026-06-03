# Angular Signals - Interview Questions

## Overview

This guide covers Angular Signals, a new reactive primitive introduced in Angular 16+. It includes comparisons with observables, computed signals, effects, and integration patterns with existing Angular features.

## Core Concept Questions

### 1. What are Angular Signals and how do they differ from Observables?

**Expected Answer:**

Signals are synchronous reactive primitives that hold values and notify consumers when those values change. Unlike Observables, Signals are always synchronous, always have a current value, and don't require subscription management.

**Detailed Comparison:**

```typescript
// Observables (RxJS)
@Component({
  selector: 'app-observable-demo',
  standalone: true,
  template: `
    <p>Count: {{ count$ | async }}</p>
    <button (click)="increment()">Increment</button>
  `
})
export class ObservableDemoComponent implements OnDestroy {
  private countSubject = new BehaviorSubject<number>(0);
  count$ = this.countSubject.asObservable();
  private subscription = new Subscription();

  ngOnInit() {
    // Need to manage subscriptions
    this.subscription.add(
      this.count$.subscribe(value => {
        console.log('Count changed:', value);
      })
    );
  }

  increment() {
    this.countSubject.next(this.countSubject.value + 1);
  }

  ngOnDestroy() {
    // Must unsubscribe to prevent memory leaks
    this.subscription.unsubscribe();
  }
}

// Signals (Angular 16+)
@Component({
  selector: 'app-signal-demo',
  standalone: true,
  template: `
    <p>Count: {{ count() }}</p>
    <button (click)="increment()">Increment</button>
  `
})
export class SignalDemoComponent {
  // No subscription management needed
  count = signal(0);

  increment() {
    // Direct, synchronous update
    this.count.update(value => value + 1);
  }

  ngOnInit() {
    // Synchronous access to current value
    console.log('Initial count:', this.count());
    
    // Effects automatically track signal dependencies
    effect(() => {
      console.log('Count changed:', this.count());
    });
  }

  // No ngOnDestroy needed - no subscriptions to clean up
}
```

**Key Differences:**

| Feature | Signals | Observables |
|---------|---------|-------------|
| **Value Access** | Synchronous `count()` | Async via subscription or `async` pipe |
| **Always Has Value** | Yes | No (cold observables) |
| **Subscription** | Not required | Required |
| **Memory Management** | Automatic | Manual unsubscribe needed |
| **Lazy/Eager** | Eager (always has value) | Lazy (cold observables) |
| **Async Operations** | Not built-in | Built-in with operators |
| **Change Detection** | Automatic with OnPush | Requires async pipe or markForCheck |
| **Composition** | computed(), effect() | RxJS operators |
| **Glitch-Free** | Yes (consistent state) | Can have intermediate states |

**Advanced Comparison:**

```typescript
@Component({
  selector: 'app-comparison-demo',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="signals">
      <h3>Signals (Glitch-Free)</h3>
      <p>First Name: {{ firstName() }}</p>
      <p>Last Name: {{ lastName() }}</p>
      <p>Full Name: {{ fullName() }}</p>
      <button (click)="updateSignalName()">Update Name</button>
    </div>

    <div class="observables">
      <h3>Observables (Potential Glitches)</h3>
      <p>First Name: {{ firstName$ | async }}</p>
      <p>Last Name: {{ lastName$ | async }}</p>
      <p>Full Name: {{ fullName$ | async }}</p>
      <button (click)="updateObservableName()">Update Name</button>
    </div>
  `
})
export class ComparisonDemoComponent {
  // Signals - glitch-free updates
  firstName = signal('John');
  lastName = signal('Doe');
  fullName = computed(() => `${this.firstName()} ${this.lastName()}`);

  updateSignalName() {
    // Both updates happen atomically
    this.firstName.set('Jane');
    this.lastName.set('Smith');
    // fullName() is guaranteed to be 'Jane Smith', never 'Jane Doe' or 'John Smith'
  }

  // Observables - potential for glitches
  private firstNameSubject = new BehaviorSubject('John');
  private lastNameSubject = new BehaviorSubject('Doe');
  firstName$ = this.firstNameSubject.asObservable();
  lastName$ = this.lastNameSubject.asObservable();
  fullName$ = combineLatest([this.firstName$, this.lastName$]).pipe(
    map(([first, last]) => `${first} ${last}`)
  );

  updateObservableName() {
    // These updates happen sequentially
    this.firstNameSubject.next('Jane');
    // At this moment, fullName$ might emit 'Jane Doe'
    this.lastNameSubject.next('Smith');
    // Now fullName$ emits 'Jane Smith'
    // Subscribers might see intermediate 'Jane Doe' state (glitch)
  }
}
```

**When to Use Each:**

Use Signals when:
- Working with synchronous state
- Need simple, direct value access
- Want automatic change detection with OnPush
- Building reactive local component state
- Need glitch-free consistency

Use Observables when:
- Handling async operations (HTTP, WebSocket)
- Need time-based operations (debounce, throttle, interval)
- Require complex operators (switchMap, mergeMap, catchError)
- Working with event streams
- Need cancellation capabilities

**Follow-up Questions:**
1. Can Signals and Observables be used together?
2. How do Signals affect change detection?
3. What is the "glitch-free" guarantee in Signals?

**What Interviewers Look For:**
- Understanding of reactive programming concepts
- Knowledge of when to use Signals vs Observables
- Awareness of subscription management differences
- Understanding of change detection implications

### 2. What are computed signals? How do they differ from regular signals?

**Expected Answer:**

Computed signals are derived values that automatically recalculate when their dependencies change. They are read-only and memoized for performance.

**Comprehensive Examples:**

```typescript
// Basic computed signal
@Component({
  selector: 'app-computed-demo',
  standalone: true,
  template: `
    <h3>Shopping Cart</h3>
    <p>Items: {{ items().length }}</p>
    <p>Subtotal: {{ subtotal() | currency }}</p>
    <p>Tax (8%): {{ tax() | currency }}</p>
    <p>Total: {{ total() | currency }}</p>
    
    <button (click)="addItem()">Add Item</button>
  `
})
export class ComputedDemoComponent {
  // Writable signals
  items = signal<CartItem[]>([
    { id: 1, name: 'Item 1', price: 10, quantity: 2 },
    { id: 2, name: 'Item 2', price: 20, quantity: 1 }
  ]);

  // Computed signals - automatically recalculate when items change
  subtotal = computed(() => {
    console.log('Calculating subtotal'); // Only called when items change
    return this.items().reduce((sum, item) => sum + (item.price * item.quantity), 0);
  });

  tax = computed(() => {
    console.log('Calculating tax'); // Only called when subtotal changes
    return this.subtotal() * 0.08;
  });

  total = computed(() => {
    console.log('Calculating total'); // Only called when subtotal or tax changes
    return this.subtotal() + this.tax();
  });

  addItem() {
    const newItem = {
      id: this.items().length + 1,
      name: `Item ${this.items().length + 1}`,
      price: Math.random() * 50,
      quantity: 1
    };
    
    // Triggers recalculation of all dependent computed signals
    this.items.update(items => [...items, newItem]);
  }
}

// Complex computed signals with multiple dependencies
@Component({
  selector: 'app-filtering-demo',
  standalone: true,
  template: `
    <input [(ngModel)]="searchTerm" placeholder="Search...">
    
    <select [(ngModel)]="sortBy">
      <option value="name">Name</option>
      <option value="price">Price</option>
      <option value="rating">Rating</option>
    </select>

    <select [(ngModel)]="filterCategory">
      <option value="all">All Categories</option>
      <option value="electronics">Electronics</option>
      <option value="books">Books</option>
    </select>

    <p>Showing {{ filteredAndSortedProducts().length }} products</p>

    <div *ngFor="let product of filteredAndSortedProducts()">
      {{ product.name }} - {{ product.price | currency }}
    </div>
  `
})
export class FilteringDemoComponent {
  // Source data
  allProducts = signal<Product[]>([...]);

  // Filter/sort parameters
  searchTerm = signal('');
  sortBy = signal<'name' | 'price' | 'rating'>('name');
  filterCategory = signal<string>('all');

  // Step 1: Filter by category
  categoryFiltered = computed(() => {
    const category = this.filterCategory();
    if (category === 'all') return this.allProducts();
    return this.allProducts().filter(p => p.category === category);
  });

  // Step 2: Filter by search term
  searchFiltered = computed(() => {
    const term = this.searchTerm().toLowerCase();
    if (!term) return this.categoryFiltered();
    return this.categoryFiltered().filter(p => 
      p.name.toLowerCase().includes(term)
    );
  });

  // Step 3: Sort results
  filteredAndSortedProducts = computed(() => {
    const products = [...this.searchFiltered()];
    const sortField = this.sortBy();
    
    return products.sort((a, b) => {
      if (a[sortField] < b[sortField]) return -1;
      if (a[sortField] > b[sortField]) return 1;
      return 0;
    });
  });

  // Computed signal for statistics
  stats = computed(() => ({
    total: this.allProducts().length,
    filtered: this.filteredAndSortedProducts().length,
    averagePrice: this.filteredAndSortedProducts().reduce(
      (sum, p) => sum + p.price, 0
    ) / this.filteredAndSortedProducts().length || 0
  }));
}

// Computed signals with complex logic
@Component({
  selector: 'app-form-validation',
  standalone: true,
  template: `
    <form>
      <input [(ngModel)]="email" placeholder="Email">
      <input [(ngModel)]="password" placeholder="Password">
      <input [(ngModel)]="confirmPassword" placeholder="Confirm Password">
      
      <div *ngIf="!isEmailValid()" class="error">
        Invalid email format
      </div>
      
      <div *ngIf="!isPasswordValid()" class="error">
        Password must be at least 8 characters
      </div>
      
      <div *ngIf="!doPasswordsMatch()" class="error">
        Passwords don't match
      </div>

      <button [disabled]="!isFormValid()">Submit</button>
    </form>

    <div>Form Status: {{ formStatus() }}</div>
  `
})
export class FormValidationComponent {
  email = signal('');
  password = signal('');
  confirmPassword = signal('');

  // Individual validation rules
  isEmailValid = computed(() => {
    const email = this.email();
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  });

  isPasswordValid = computed(() => {
    return this.password().length >= 8;
  });

  doPasswordsMatch = computed(() => {
    return this.password() === this.confirmPassword() && 
           this.confirmPassword().length > 0;
  });

  // Combined validation
  isFormValid = computed(() => {
    return this.isEmailValid() && 
           this.isPasswordValid() && 
           this.doPasswordsMatch();
  });

  // Computed status message
  formStatus = computed(() => {
    if (this.isFormValid()) return 'Ready to submit';
    if (!this.email()) return 'Please enter email';
    if (!this.isEmailValid()) return 'Invalid email';
    if (!this.password()) return 'Please enter password';
    if (!this.isPasswordValid()) return 'Password too short';
    if (!this.doPasswordsMatch()) return 'Passwords must match';
    return 'Please complete the form';
  });
}

// Performance: Computed signals are memoized
@Component({
  selector: 'app-memoization-demo',
  standalone: true,
  template: `
    <p>Expensive Result: {{ expensiveComputation() }}</p>
    <p>Another Reference: {{ expensiveComputation() }}</p>
    <p>Third Reference: {{ expensiveComputation() }}</p>
    
    <button (click)="unrelatedUpdate()">Update Unrelated</button>
    <button (click)="relatedUpdate()">Update Related</button>
  `
})
export class MemoizationDemoComponent {
  sourceData = signal([1, 2, 3, 4, 5]);
  unrelatedData = signal('something');

  expensiveComputation = computed(() => {
    console.log('EXPENSIVE COMPUTATION RUNNING');
    // This only runs when sourceData changes, not on every access
    return this.sourceData().reduce((sum, n) => sum + n * n, 0);
  });

  unrelatedUpdate() {
    // This does NOT trigger expensiveComputation
    this.unrelatedData.set('updated');
  }

  relatedUpdate() {
    // This DOES trigger expensiveComputation
    this.sourceData.update(arr => [...arr, arr.length + 1]);
  }
}
```

**Key Characteristics:**

1. **Read-Only**: Cannot directly set computed signal values
2. **Lazy Evaluation**: Only recalculates when accessed and dependencies changed
3. **Memoized**: Returns cached value if dependencies haven't changed
4. **Automatic Dependencies**: Tracks signals used during computation
5. **Glitch-Free**: Updates atomically with all dependencies

**Follow-up Questions:**
1. How does Angular track computed signal dependencies?
2. Can computed signals depend on other computed signals?
3. What happens if a computed signal's computation throws an error?

**What Interviewers Look For:**
- Understanding of memoization and performance benefits
- Knowledge of dependency tracking
- Ability to design efficient computed signal hierarchies
- Understanding of when to break computations into multiple computed signals

### 3. What are effects in Angular Signals? When should you use them?

**Expected Answer:**

Effects are operations that run when signals they depend on change. They're used for side effects like logging, analytics, localStorage sync, or DOM manipulation.

**Comprehensive Examples:**

```typescript
// Basic effect usage
@Component({
  selector: 'app-effect-demo',
  standalone: true
})
export class EffectDemoComponent {
  count = signal(0);
  user = signal({ name: 'John', age: 30 });

  constructor() {
    // Effect automatically tracks dependencies
    effect(() => {
      console.log('Count changed to:', this.count());
    });

    // Effect with multiple dependencies
    effect(() => {
      console.log(`User: ${this.user().name}, Count: ${this.count()}`);
      // Runs when either user or count changes
    });

    // Effect with cleanup
    effect((onCleanup) => {
      const interval = setInterval(() => {
        console.log('Current count:', this.count());
      }, 1000);

      // Cleanup function runs before next effect execution or on destroy
      onCleanup(() => {
        clearInterval(interval);
        console.log('Cleaning up interval');
      });
    });
  }

  increment() {
    this.count.update(c => c + 1);
  }
}

// Syncing with localStorage
@Component({
  selector: 'app-storage-sync',
  standalone: true,
  template: `
    <input [(ngModel)]="username" placeholder="Username">
    <input [(ngModel)]="theme" placeholder="Theme">
    
    <p>Settings will persist across page reloads</p>
  `
})
export class StorageSyncComponent {
  username = signal(localStorage.getItem('username') || '');
  theme = signal(localStorage.getItem('theme') || 'light');

  constructor() {
    // Sync username to localStorage
    effect(() => {
      const name = this.username();
      if (name) {
        localStorage.setItem('username', name);
      } else {
        localStorage.removeItem('username');
      }
    });

    // Sync theme to localStorage
    effect(() => {
      const currentTheme = this.theme();
      localStorage.setItem('theme', currentTheme);
      document.body.className = currentTheme;
    });
  }
}

// Advanced: Effect with manual dependency tracking
@Component({
  selector: 'app-manual-tracking',
  standalone: true
})
export class ManualTrackingComponent {
  counter = signal(0);
  autoSave = signal(true);

  constructor() {
    // Effect only tracks autoSave, not counter
    effect(() => {
      if (this.autoSave()) {
        // Use untracked() to read without creating dependency
        const currentCount = untracked(() => this.counter());
        console.log('Auto-saving count:', currentCount);
        this.saveToServer(currentCount);
      }
    });
  }

  private saveToServer(count: number) {
    // Save logic
  }
}

// Effect for analytics
@Component({
  selector: 'app-analytics',
  standalone: true
})
export class AnalyticsComponent implements OnInit {
  currentRoute = signal('');
  userId = signal('');
  sessionDuration = signal(0);

  constructor(
    private router: Router,
    private analytics: AnalyticsService
  ) {}

  ngOnInit() {
    // Track route changes
    effect(() => {
      const route = this.currentRoute();
      const user = this.userId();
      
      if (route && user) {
        this.analytics.trackPageView({
          page: route,
          userId: user,
          timestamp: new Date()
        });
      }
    });

    // Track session duration
    effect((onCleanup) => {
      const startTime = Date.now();
      
      const interval = setInterval(() => {
        this.sessionDuration.set(Math.floor((Date.now() - startTime) / 1000));
      }, 1000);

      onCleanup(() => {
        clearInterval(interval);
        this.analytics.trackSessionDuration({
          userId: this.userId(),
          duration: this.sessionDuration()
        });
      });
    });
  }
}

// Effect for WebSocket synchronization
@Component({
  selector: 'app-websocket-sync',
  standalone: true
})
export class WebSocketSyncComponent {
  messages = signal<Message[]>([]);
  connectionStatus = signal<'connected' | 'disconnected'>('disconnected');
  roomId = signal('');

  constructor(private wsService: WebSocketService) {}

  ngOnInit() {
    // Connect/disconnect based on roomId
    effect((onCleanup) => {
      const room = this.roomId();
      
      if (!room) return;

      console.log('Connecting to room:', room);
      const connection = this.wsService.connect(room);

      connection.subscribe({
        next: (message) => {
          this.messages.update(msgs => [...msgs, message]);
        },
        error: () => {
          this.connectionStatus.set('disconnected');
        },
        complete: () => {
          this.connectionStatus.set('disconnected');
        }
      });

      this.connectionStatus.set('connected');

      onCleanup(() => {
        console.log('Disconnecting from room:', room);
        connection.unsubscribe();
        this.connectionStatus.set('disconnected');
      });
    });
  }
}

// Effect options
@Component({
  selector: 'app-effect-options',
  standalone: true
})
export class EffectOptionsComponent {
  data = signal('');

  constructor() {
    // Allow signal writes within effect (use with caution)
    effect(() => {
      const data = this.data();
      if (data.length > 100) {
        // This would normally cause an error, but allowSignalWrites permits it
        this.data.set(data.substring(0, 100));
      }
    }, { allowSignalWrites: true });

    // Manual effect with explicit injector
    const injector = inject(Injector);
    const manualEffect = effect(() => {
      console.log('Manual effect:', this.data());
    }, { injector, manualCleanup: true });

    // Later: manually destroy
    // manualEffect.destroy();
  }
}
```

**When to Use Effects:**

Good Use Cases:
- Syncing state to localStorage/sessionStorage
- Logging and analytics
- WebSocket subscriptions
- DOM manipulation (rarely needed)
- Triggering animations
- Debugging (development only)

Avoid Using Effects For:
- Deriving values (use computed instead)
- Updating other signals (can cause infinite loops)
- HTTP requests (use observables instead)
- Complex business logic (use methods)

**Effect vs Computed:**

```typescript
@Component({
  selector: 'app-effect-vs-computed',
  standalone: true
})
export class EffectVsComputedComponent {
  firstName = signal('John');
  lastName = signal('Doe');

  // GOOD: Use computed for derived values
  fullName = computed(() => `${this.firstName()} ${this.lastName()}`);

  // BAD: Don't use effect to derive values
  fullNameBad = signal('');
  
  constructor() {
    // Anti-pattern: Using effect for derived value
    effect(() => {
      this.fullNameBad.set(`${this.firstName()} ${this.lastName()}`);
    }, { allowSignalWrites: true });
    // This is inefficient and can cause issues

    // GOOD: Effect for side effects
    effect(() => {
      console.log('Name changed:', this.fullName());
    });
  }
}
```

**Follow-up Questions:**
1. How do you prevent infinite loops in effects?
2. What's the cleanup function in effects used for?
3. Can effects be created outside of a component constructor?

**What Interviewers Look For:**
- Understanding of appropriate use cases for effects
- Knowledge of effect cleanup
- Awareness of potential pitfalls (infinite loops)
- Understanding of effect vs computed distinction

### 4. How do Signals integrate with OnPush change detection?

**Expected Answer:**

Signals work seamlessly with OnPush change detection because Angular automatically marks components for check when signals used in templates change.

**Implementation:**

```typescript
// Without Signals - OnPush requires manual markForCheck
@Component({
  selector: 'app-onpush-manual',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count: {{ count }}</p>
    <button (click)="increment()">Increment</button>
  `
})
export class OnPushManualComponent {
  count = 0;

  constructor(private cdr: ChangeDetectorRef) {}

  increment() {
    this.count++;
    // OnPush won't detect this change without manual trigger
    this.cdr.markForCheck(); // Required!
  }
}

// With Signals - OnPush works automatically
@Component({
  selector: 'app-onpush-signals',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count: {{ count() }}</p>
    <button (click)="increment()">Increment</button>
  `
})
export class OnPushSignalsComponent {
  count = signal(0);

  increment() {
    this.count.update(c => c + 1);
    // No markForCheck needed - Angular handles it automatically!
  }
}

// Complex example with nested components
@Component({
  selector: 'app-parent',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ChildComponent],
  template: `
    <h2>Parent: {{ parentCount() }}</h2>
    <button (click)="incrementParent()">Increment Parent</button>
    
    <app-child [count]="parentCount()" />
  `
})
export class ParentComponent {
  parentCount = signal(0);

  incrementParent() {
    this.parentCount.update(c => c + 1);
    // Child component automatically updates due to signal change
  }
}

@Component({
  selector: 'app-child',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h3>Child: {{ count }}</h3>
    <p>Doubled: {{ doubled() }}</p>
  `
})
export class ChildComponent {
  @Input() count = 0;
  
  // Computed signal works with OnPush
  doubled = computed(() => this.count * 2);
}

// Signal-based service with OnPush
@Injectable({ providedIn: 'root' })
export class CounterService {
  private countSignal = signal(0);
  count = this.countSignal.asReadonly();

  increment() {
    this.countSignal.update(c => c + 1);
  }
}

@Component({
  selector: 'app-service-consumer',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count from service: {{ counterService.count() }}</p>
    <button (click)="counterService.increment()">Increment</button>
  `
})
export class ServiceConsumerComponent {
  constructor(public counterService: CounterService) {}
  // No manual change detection needed - signals handle it!
}
```

**Follow-up Questions:**
1. Do signals work with zone.js or zoneless change detection?
2. How do signals affect performance compared to traditional OnPush?
3. Can you mix signals and observables with OnPush?

**What Interviewers Look For:**
- Understanding of OnPush change detection
- Knowledge of how signals simplify change detection
- Awareness of performance benefits
- Understanding of signal-based architecture

### 5. When should you use Signals vs Observables in an Angular application?

**Expected Answer:**

Use Signals for synchronous local state and derived values. Use Observables for async operations, streams, and complex transformations.

**Decision Matrix:**

```typescript
// Use Signals for: Local Component State
@Component({
  selector: 'app-form',
  standalone: true,
  template: `
    <input [value]="username()" (input)="updateUsername($event)">
    <p>Characters: {{ username().length }}</p>
    <p>Valid: {{ isValid() }}</p>
  `
})
export class FormComponent {
  username = signal('');
  isValid = computed(() => this.username().length >= 3);

  updateUsername(event: Event) {
    this.username.set((event.target as HTMLInputElement).value);
  }
}

// Use Observables for: HTTP Requests
@Component({
  selector: 'app-data-fetcher',
  standalone: true,
  template: `
    <div *ngIf="users$ | async as users">
      <div *ngFor="let user of users">{{ user.name }}</div>
    </div>
  `
})
export class DataFetcherComponent {
  users$ = this.http.get<User[]>('/api/users').pipe(
    catchError(error => {
      console.error(error);
      return of([]);
    })
  );

  constructor(private http: HttpClient) {}
}

// Hybrid Approach: Convert Observable to Signal
@Component({
  selector: 'app-hybrid',
  standalone: true,
  template: `
    <div *ngIf="users() as users">
      <div *ngFor="let user of users">{{ user.name }}</div>
    </div>
    
    <p>Count: {{ userCount() }}</p>
  `
})
export class HybridComponent {
  // Observable for HTTP
  private users$ = this.http.get<User[]>('/api/users');
  
  // Convert to signal using toSignal
  users = toSignal(this.users$, { initialValue: [] });
  
  // Computed signal from converted observable
  userCount = computed(() => this.users().length);

  constructor(private http: HttpClient) {}
}

// Use Observables for: Event Streams with Operators
@Component({
  selector: 'app-search',
  standalone: true,
  template: `
    <input #searchInput placeholder="Search...">
    <div *ngFor="let result of searchResults$ | async">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent implements AfterViewInit {
  @ViewChild('searchInput') searchInput!: ElementRef;
  searchResults$!: Observable<SearchResult[]>;

  constructor(private searchService: SearchService) {}

  ngAfterViewInit() {
    this.searchResults$ = fromEvent(this.searchInput.nativeElement, 'input').pipe(
      map(event => (event.target as HTMLInputElement).value),
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => this.searchService.search(term)),
      catchError(() => of([]))
    );
  }
}

// Use Signals for: Derived State
@Component({
  selector: 'app-cart',
  standalone: true,
  template: `
    <p>Items: {{ items().length }}</p>
    <p>Total: {{ total() | currency }}</p>
    <p>Discount: {{ discount() | currency }}</p>
    <p>Final: {{ finalTotal() | currency }}</p>
  `
})
export class CartComponent {
  items = signal<CartItem[]>([]);
  discountCode = signal('');

  total = computed(() => 
    this.items().reduce((sum, item) => sum + item.price, 0)
  );

  discount = computed(() => {
    const code = this.discountCode();
    if (code === 'SAVE10') return this.total() * 0.1;
    if (code === 'SAVE20') return this.total() * 0.2;
    return 0;
  });

  finalTotal = computed(() => this.total() - this.discount());
}

// Combined: Signals + Observables
@Component({
  selector: 'app-combined',
  standalone: true
})
export class CombinedComponent {
  // Signal for local state
  selectedUserId = signal<string | null>(null);

  // Observable that reacts to signal changes
  selectedUser$ = toObservable(this.selectedUserId).pipe(
    filter(id => id !== null),
    switchMap(id => this.http.get<User>(`/api/users/${id}`)),
    catchError(() => of(null))
  );

  // Convert back to signal for template
  selectedUser = toSignal(this.selectedUser$);

  constructor(private http: HttpClient) {}

  selectUser(id: string) {
    this.selectedUserId.set(id);
  }
}
```

**Decision Guide:**

| Scenario | Use |
|----------|-----|
| Local component state | Signals |
| Derived/computed values | Signals (computed) |
| Form inputs | Signals |
| HTTP requests | Observables |
| WebSocket streams | Observables |
| Debouncing/throttling | Observables |
| Error handling & retry | Observables |
| Complex async flows | Observables |
| Time-based operations | Observables |
| Side effects | Signals (effects) |

**Follow-up Questions:**
1. How do you convert between signals and observables?
2. Can you use RxJS operators with signals?
3. What's the performance difference between signals and observables?

**What Interviewers Look For:**
- Pragmatic understanding of both approaches
- Knowledge of interop functions (toSignal, toObservable)
- Ability to choose the right tool for the job
- Understanding of hybrid patterns

## Key Takeaways

1. **Signals vs Observables**: Signals are synchronous reactive primitives always holding current values, while Observables are asynchronous streams requiring subscription management.

2. **Computed Signals**: Read-only derived values that automatically recalculate when dependencies change, with memoization for performance.

3. **Effects for Side Effects**: Use effects for side effects like logging, storage sync, and analytics - not for deriving values (use computed instead).

4. **Glitch-Free Consistency**: Signals guarantee consistent state across all computed values during updates, preventing intermediate states.

5. **OnPush Integration**: Signals work seamlessly with OnPush change detection, automatically marking components for check without manual markForCheck calls.

6. **toSignal and toObservable**: Angular provides interop functions to convert between signals and observables for hybrid patterns.

7. **Effect Cleanup**: Effects support cleanup functions for resource management, running before next execution or component destruction.

8. **Signal Mutations**: Use set() for replacing values, update() for transforming current value, and mutate() for in-place updates of objects/arrays.

9. **Readonly Signals**: Use asReadonly() to expose signals from services while preventing external modifications.

10. **When to Choose**: Use Signals for synchronous state and derived values; use Observables for async operations, streams, and complex transformations with RxJS operators.

## Red Flags to Avoid

- Using effects to derive values instead of computed signals
- Creating circular dependencies between signals
- Writing to signals within effects without allowSignalWrites
- Using signals for async operations instead of observables
- Not understanding the difference between set(), update(), and mutate()
- Overusing signals for everything when observables are more appropriate
- Not cleaning up resources in effects with cleanup function
- Using untracked() without understanding dependency implications
- Mixing signal and non-signal state in confusing ways
- Not leveraging OnPush with signals for performance

## Interview Preparation Tips

- Practice converting existing code from observables to signals
- Understand the glitch-free guarantee and be ready to explain it
- Know when to use computed vs effects vs regular methods
- Be familiar with toSignal and toObservable interop functions
- Prepare examples of real-world signal usage
- Understand how signals enable zoneless change detection
- Know the signal API: set, update, mutate, asReadonly
- Be ready to discuss performance benefits of signals
- Understand effect cleanup and resource management
- Practice combining signals and observables in hybrid patterns
