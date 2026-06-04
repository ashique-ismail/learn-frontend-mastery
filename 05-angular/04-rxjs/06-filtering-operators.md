# RxJS Filtering Operators

## Table of Contents
- [Introduction](#introduction)
- [filter](#filter)
- [debounceTime](#debouncetime)
- [distinctUntilChanged](#distinctuntilchanged)
- [take](#take)
- [takeUntil](#takeuntil)
- [takeWhile](#takewhile)
- [skip](#skip)
- [first and last](#first-and-last)
- [throttleTime](#throttletime)
- [audit and sample](#audit-and-sample)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Filtering operators allow you to selectively emit values from an observable based on various criteria. These operators are essential for controlling data flow, improving performance, and creating efficient reactive patterns in Angular applications. Understanding when and how to use each filtering operator is crucial for building responsive, performant applications.

## filter

The `filter` operator emits only values that pass a predicate function. It's the RxJS equivalent of `Array.prototype.filter()`.

### Basic Usage

```typescript
import { filter } from 'rxjs/operators';
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products$ | async">
      {{ product.name }} - ${{ product.price }}
    </div>
  `
})
export class ProductListComponent implements OnInit {
  private productService = inject(ProductService);

  // Filter products over $50
  products$ = this.productService.getProducts().pipe(
    filter(products => products.price > 50)
  );
}
```

### Type Guards with filter

```typescript
import { filter, map } from 'rxjs/operators';

interface LoadingState { status: 'loading' }
interface SuccessState { status: 'success'; data: User }
interface ErrorState { status: 'error'; error: string }

type State = LoadingState | SuccessState | ErrorState;

@Component({
  selector: 'app-user-profile'
})
export class UserProfileComponent {
  private store = inject(Store);

  // Type guard function
  isSuccess(state: State): state is SuccessState {
    return state.status === 'success';
  }

  // Filter with type narrowing
  user$ = this.store.select('userState').pipe(
    filter(this.isSuccess),
    map(state => state.data) // TypeScript knows state is SuccessState
  );
}
```

### Real-World Example: Event Filtering

```typescript
@Component({
  selector: 'app-drag-drop',
  template: `
    <div 
      (mousedown)="mouseDown$.next($event)"
      (mousemove)="mouseMove$.next($event)"
      (mouseup)="mouseUp$.next($event)"
      class="draggable">
      Drag me
    </div>
  `
})
export class DragDropComponent implements OnInit {
  mouseDown$ = new Subject<MouseEvent>();
  mouseMove$ = new Subject<MouseEvent>();
  mouseUp$ = new Subject<MouseEvent>();
  
  drag$ = this.mouseDown$.pipe(
    switchMap(() => this.mouseMove$.pipe(
      // Only emit mouse moves while button is pressed
      filter(event => event.buttons === 1),
      takeUntil(this.mouseUp$)
    ))
  );

  ngOnInit() {
    this.drag$.subscribe(event => {
      console.log('Dragging at:', event.clientX, event.clientY);
    });
  }
}
```

### Filtering Null/Undefined Values

```typescript
import { filter, map } from 'rxjs/operators';

@Injectable()
export class UserService {
  private userSubject = new BehaviorSubject<User | null>(null);
  
  // Only emit when user exists
  user$ = this.userSubject.asObservable().pipe(
    filter((user): user is User => user !== null && user !== undefined)
  );

  // Alternative: using Boolean constructor
  userAlt$ = this.userSubject.asObservable().pipe(
    filter(Boolean) // Filters out null, undefined, false, 0, ''
  ) as Observable<User>;
}
```

## debounceTime

`debounceTime` delays emissions until a specified time has passed without another emission. Essential for search inputs and form validation.

### Search Input Debouncing

```typescript
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-search',
  template: `
    <input 
      [formControl]="searchControl" 
      placeholder="Search products..."
    >
    <div *ngFor="let result of searchResults$ | async">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent {
  private searchService = inject(SearchService);
  
  searchControl = new FormControl('');

  searchResults$ = this.searchControl.valueChanges.pipe(
    debounceTime(300), // Wait 300ms after user stops typing
    distinctUntilChanged(), // Only emit if value actually changed
    filter(term => term.length >= 3), // Only search if 3+ characters
    switchMap(term => this.searchService.search(term))
  );
}
```

### Auto-Save with Debounce

```typescript
@Component({
  selector: 'app-editor',
  template: `
    <textarea 
      [formControl]="contentControl"
      placeholder="Start typing..."
    ></textarea>
    <div *ngIf="saveStatus$ | async as status">
      {{ status }}
    </div>
  `
})
export class EditorComponent implements OnInit {
  private documentService = inject(DocumentService);
  
  contentControl = new FormControl('');
  saveStatus$ = new BehaviorSubject<string>('');

  ngOnInit() {
    this.contentControl.valueChanges.pipe(
      debounceTime(2000), // Wait 2 seconds after user stops typing
      distinctUntilChanged(),
      tap(() => this.saveStatus$.next('Saving...')),
      switchMap(content => 
        this.documentService.save(content).pipe(
          map(() => 'Saved'),
          catchError(() => of('Save failed'))
        )
      )
    ).subscribe(status => this.saveStatus$.next(status));
  }
}
```

### Form Validation with Debounce

```typescript
@Component({
  selector: 'app-signup'
})
export class SignupComponent implements OnInit {
  private userService = inject(UserService);
  
  usernameControl = new FormControl('');
  usernameAvailable$ = new BehaviorSubject<boolean | null>(null);
  checkingUsername$ = new BehaviorSubject<boolean>(false);

  ngOnInit() {
    this.usernameControl.valueChanges.pipe(
      debounceTime(500), // Wait for user to stop typing
      distinctUntilChanged(),
      filter(username => username.length >= 3),
      tap(() => {
        this.checkingUsername$.next(true);
        this.usernameAvailable$.next(null);
      }),
      switchMap(username => 
        this.userService.checkUsernameAvailability(username).pipe(
          map(available => ({ available, username })),
          catchError(() => of({ available: false, username }))
        )
      ),
      tap(() => this.checkingUsername$.next(false))
    ).subscribe(({ available }) => {
      this.usernameAvailable$.next(available);
    });
  }
}
```

## distinctUntilChanged

`distinctUntilChanged` only emits when the current value is different from the previous value.

### Basic Usage

```typescript
import { distinctUntilChanged } from 'rxjs/operators';

@Component({
  selector: 'app-filter-panel'
})
export class FilterPanelComponent {
  categoryControl = new FormControl('');

  // Only trigger API call when category actually changes
  categoryChanges$ = this.categoryControl.valueChanges.pipe(
    distinctUntilChanged(),
    tap(category => console.log('Category changed to:', category))
  );
}
```

### Custom Comparison Function

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
  lastModified: Date;
}

@Component({
  selector: 'app-product-monitor'
})
export class ProductMonitorComponent {
  private productService = inject(ProductService);

  // Only emit when product ID or price changes (ignore lastModified)
  productChanges$ = this.productService.productUpdates$.pipe(
    distinctUntilChanged((prev, curr) =>
      prev.id === curr.id && prev.price === curr.price
    )
  );
}
```

### Deep Comparison with Keys

```typescript
import { distinctUntilChanged, map } from 'rxjs/operators';

interface UserSettings {
  theme: string;
  language: string;
  notifications: boolean;
  lastLogin: Date; // We don't care about this changing
}

@Component({
  selector: 'app-settings'
})
export class SettingsComponent {
  private store = inject(Store);

  // Only emit when relevant settings change
  settings$ = this.store.select('userSettings').pipe(
    distinctUntilChanged((prev, curr) =>
      prev.theme === curr.theme &&
      prev.language === curr.language &&
      prev.notifications === curr.notifications
    )
  );
}
```

### With Key Selector

```typescript
import { distinctUntilKeyChanged } from 'rxjs/operators';

interface CartItem {
  productId: string;
  quantity: number;
  price: number;
}

@Component({
  selector: 'app-cart'
})
export class CartComponent {
  private cartService = inject(CartService);

  // Only emit when quantity changes for an item
  cartUpdates$ = this.cartService.cartItems$.pipe(
    distinctUntilKeyChanged('quantity'),
    tap(item => console.log('Quantity changed:', item.quantity))
  );
}
```

## take

`take` emits only the first n values, then completes.

### Basic Usage

```typescript
import { take } from 'rxjs/operators';

@Component({
  selector: 'app-welcome'
})
export class WelcomeComponent implements OnInit {
  private router = inject(Router);

  ngOnInit() {
    // Only show welcome message on first visit
    this.showWelcomeMessage$.pipe(
      take(1)
    ).subscribe(show => {
      if (show) {
        this.displayWelcome();
      }
    });
  }
}
```

### One-Time Data Fetch

```typescript
@Component({
  selector: 'app-config'
})
export class ConfigComponent {
  private configService = inject(ConfigService);

  // Get config once, then complete
  config$ = this.configService.config$.pipe(
    take(1),
    tap(config => console.log('Config loaded:', config))
  );
}
```

### Limited Retry Attempts

```typescript
@Injectable()
export class ResilientHttpService {
  constructor(private http: HttpClient) {}

  fetchWithRetry<T>(url: string, maxAttempts: number = 3): Observable<T> {
    return defer(() => this.http.get<T>(url)).pipe(
      retry({
        count: maxAttempts - 1,
        delay: 1000
      }),
      take(1) // Ensure we only get one successful response
    );
  }
}
```

## takeUntil

`takeUntil` emits values until a notifier observable emits, then completes. Essential for preventing memory leaks.

### Component Cleanup Pattern

```typescript
import { takeUntil } from 'rxjs/operators';
import { Subject } from 'rxjs';

@Component({
  selector: 'app-data-viewer'
})
export class DataViewerComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private dataService = inject(DataService);

  ngOnInit() {
    // All subscriptions automatically cleaned up on destroy
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      console.log('Data:', data);
    });

    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(count => {
      console.log('Tick:', count);
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Modern takeUntilDestroyed (Angular 16+)

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-modern-cleanup'
})
export class ModernCleanupComponent implements OnInit {
  private dataService = inject(DataService);
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    // Automatically unsubscribes when component is destroyed
    this.dataService.getData().pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(data => {
      console.log('Data:', data);
    });
  }
}

// Or in constructor (no need for DestroyRef)
@Component({
  selector: 'app-auto-cleanup'
})
export class AutoCleanupComponent {
  private dataService = inject(DataService);

  data$ = this.dataService.getData().pipe(
    takeUntilDestroyed() // Uses injection context
  );
}
```

### Cancel In-Progress Operations

```typescript
@Component({
  selector: 'app-search-with-cancel',
  template: `
    <input [formControl]="searchControl">
    <button (click)="cancel$.next()">Cancel</button>
    <div *ngFor="let result of results$ | async">{{ result }}</div>
  `
})
export class SearchWithCancelComponent {
  private searchService = inject(SearchService);
  
  searchControl = new FormControl('');
  cancel$ = new Subject<void>();

  results$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term =>
      this.searchService.search(term).pipe(
        takeUntil(this.cancel$) // Cancel current search
      )
    )
  );
}
```

## takeWhile

`takeWhile` emits values while a predicate is true, then completes.

### Conditional Streaming

```typescript
interface GameState {
  lives: number;
  score: number;
  level: number;
}

@Component({
  selector: 'app-game'
})
export class GameComponent {
  private gameService = inject(GameService);

  // Stream game state while player has lives
  gameState$ = this.gameService.state$.pipe(
    takeWhile(state => state.lives > 0, true), // true = include final emission
    tap(state => {
      if (state.lives === 0) {
        console.log('Game Over! Final score:', state.score);
      }
    })
  );
}
```

### Time-Based Conditions

```typescript
@Component({
  selector: 'app-timer-trial'
})
export class TimerTrialComponent implements OnInit {
  private startTime = Date.now();
  private trialDuration = 60000; // 60 seconds

  ngOnInit() {
    interval(1000).pipe(
      takeWhile(() => Date.now() - this.startTime < this.trialDuration),
      map(() => Math.floor((this.trialDuration - (Date.now() - this.startTime)) / 1000))
    ).subscribe({
      next: secondsLeft => console.log('Time left:', secondsLeft),
      complete: () => console.log('Trial expired')
    });
  }
}
```

## skip

`skip` ignores the first n emissions.

### Skip Initial Values

```typescript
@Component({
  selector: 'app-form-tracker'
})
export class FormTrackerComponent {
  form = new FormGroup({
    name: new FormControl(''),
    email: new FormControl('')
  });

  // Skip the initial valueChanges emission
  formChanges$ = this.form.valueChanges.pipe(
    skip(1), // Skip the first emission
    tap(() => console.log('Form was modified by user'))
  );
}
```

### Skip Loading States

```typescript
@Component({
  selector: 'app-data-grid'
})
export class DataGridComponent {
  private dataService = inject(DataService);

  // Skip initial loading state
  data$ = this.dataService.dataWithLoadingState$.pipe(
    skip(1), // Skip initial { loading: true, data: null }
    filter(state => !state.loading)
  );
}
```

## first and last

`first` emits the first value (optionally matching a predicate) then completes. `last` emits the last value before the source completes.

### first Operator

```typescript
import { first } from 'rxjs/operators';

@Component({
  selector: 'app-user-loader'
})
export class UserLoaderComponent {
  private userService = inject(UserService);

  // Get first logged-in user, then complete
  loggedInUser$ = this.userService.users$.pipe(
    first(user => user.isLoggedIn),
    tap(user => console.log('First logged-in user:', user))
  );

  // Just get the very first emission
  initialUser$ = this.userService.users$.pipe(
    first()
  );
}
```

### last Operator

```typescript
import { last } from 'rxjs/operators';

@Component({
  selector: 'app-batch-processor'
})
export class BatchProcessorComponent {
  processBatch(items: string[]): Observable<string> {
    return from(items).pipe(
      concatMap(item => this.processItem(item)),
      last(), // Get the last processed item
      tap(finalItem => console.log('Last item processed:', finalItem))
    );
  }

  private processItem(item: string): Observable<string> {
    return of(`Processed: ${item}`).pipe(delay(100));
  }
}
```

## throttleTime

`throttleTime` emits the first value, then ignores subsequent values for a specified duration.

### Scroll Event Throttling

```typescript
import { throttleTime } from 'rxjs/operators';
import { fromEvent } from 'rxjs';

@Component({
  selector: 'app-infinite-scroll'
})
export class InfiniteScrollComponent implements OnInit {
  private elementRef = inject(ElementRef);

  ngOnInit() {
    fromEvent(window, 'scroll').pipe(
      throttleTime(200), // Emit at most once every 200ms
      map(() => window.scrollY),
      filter(scrollY => this.isNearBottom(scrollY))
    ).subscribe(() => {
      this.loadMoreItems();
    });
  }

  private isNearBottom(scrollY: number): boolean {
    const threshold = 100;
    return window.innerHeight + scrollY >= document.body.offsetHeight - threshold;
  }

  private loadMoreItems() {
    console.log('Loading more items...');
  }
}
```

### Button Click Throttling

```typescript
@Component({
  selector: 'app-rate-limited-action',
  template: `
    <button (click)="action$.next($event)">
      Submit (max once per 2 seconds)
    </button>
  `
})
export class RateLimitedActionComponent implements OnInit {
  action$ = new Subject<Event>();

  ngOnInit() {
    this.action$.pipe(
      throttleTime(2000), // Allow one click every 2 seconds
      tap(() => console.log('Action executed'))
    ).subscribe(() => {
      this.performAction();
    });
  }

  private performAction() {
    console.log('Performing rate-limited action');
  }
}
```

## audit and sample

`auditTime` emits the most recent value after a specified duration. `sampleTime` emits the most recent value at regular intervals.

### auditTime Example

```typescript
import { auditTime } from 'rxjs/operators';

@Component({
  selector: 'app-mouse-tracker'
})
export class MouseTrackerComponent implements OnInit {
  mouseMove$ = new Subject<MouseEvent>();

  ngOnInit() {
    this.mouseMove$.pipe(
      auditTime(1000) // Emit most recent mouse position every second
    ).subscribe(event => {
      console.log('Mouse position:', event.clientX, event.clientY);
    });
  }

  onMouseMove(event: MouseEvent) {
    this.mouseMove$.next(event);
  }
}
```

### sampleTime Example

```typescript
import { sampleTime } from 'rxjs/operators';

@Component({
  selector: 'app-performance-monitor'
})
export class PerformanceMonitorComponent implements OnInit {
  private metrics$ = new Subject<PerformanceMetrics>();

  ngOnInit() {
    // Sample metrics every 5 seconds
    this.metrics$.pipe(
      sampleTime(5000)
    ).subscribe(metrics => {
      console.log('Sampled metrics:', metrics);
      this.sendToAnalytics(metrics);
    });
  }
}
```

## Common Mistakes

### 1. Forgetting to Unsubscribe

```typescript
// WRONG: Memory leak
ngOnInit() {
  this.dataService.getData().subscribe(data => {
    this.data = data;
  });
}

// RIGHT: Use takeUntilDestroyed
data$ = this.dataService.getData().pipe(
  takeUntilDestroyed()
);
```

### 2. Using debounceTime for Everything

```typescript
// WRONG: Debounce not needed for button clicks
buttonClick$.pipe(
  debounceTime(300) // User might click multiple times intentionally
)

// RIGHT: Use throttleTime for user actions
buttonClick$.pipe(
  throttleTime(1000) // Prevent accidental double-clicks
)
```

### 3. Not Including Final Value with takeWhile

```typescript
// WRONG: Misses the value that fails the predicate
source$.pipe(
  takeWhile(x => x < 10) // Doesn't emit 10
)

// RIGHT: Include final value
source$.pipe(
  takeWhile(x => x < 10, true) // Emits 10 before completing
)
```

### 4. Inefficient distinctUntilChanged

```typescript
// WRONG: Creates new object reference every time
source$.pipe(
  map(data => ({ ...data })), // New object!
  distinctUntilChanged() // Always different reference
)

// RIGHT: Use custom comparator
source$.pipe(
  map(data => ({ ...data })),
  distinctUntilChanged((prev, curr) => prev.id === curr.id)
)
```

## Best Practices

### 1. Combine Filtering Operators

```typescript
searchControl.valueChanges.pipe(
  debounceTime(300),      // Wait for user to stop typing
  distinctUntilChanged(), // Only if value changed
  filter(term => term.length >= 3), // Minimum length
  takeUntilDestroyed()    // Cleanup
)
```

### 2. Use Type Guards with filter

```typescript
observable$.pipe(
  filter((value): value is NonNullable<typeof value> => value != null),
  // TypeScript knows value is not null/undefined here
  map(value => value.property)
)
```

### 3. Prefer takeUntilDestroyed (Angular 16+)

```typescript
// Modern way
data$ = this.service.getData().pipe(
  takeUntilDestroyed()
);

// Avoid manual cleanup
private destroy$ = new Subject<void>();
ngOnDestroy() { /* ... */ }
```

### 4. Choose Right Timing Operator

```typescript
// debounceTime - Wait for pause (search input)
searchInput$.pipe(debounceTime(300))

// throttleTime - Limit rate (button clicks)
buttonClick$.pipe(throttleTime(1000))

// auditTime - Most recent after period (mouse tracking)
mouseMove$.pipe(auditTime(100))

// sampleTime - Regular samples (performance monitoring)
metrics$.pipe(sampleTime(5000))
```

## Interview Questions

### Q1: What's the difference between debounceTime and throttleTime?

**Answer:**
- **debounceTime**: Waits for silence (pause in emissions) before emitting. Resets timer on each emission. Use for search inputs.
- **throttleTime**: Emits first value, then ignores subsequent values for duration. Use for limiting user actions like button clicks.

### Q2: When should you use takeUntil vs takeWhile?

**Answer:**
- **takeUntil**: Use with a notifier observable to stop when an event occurs (component destruction, cancel button).
- **takeWhile**: Use with a predicate to stop when a condition becomes false (game over, timeout).

### Q3: Why combine distinctUntilChanged with debounceTime?

**Answer:** `debounceTime` alone will still emit if the user types the same value again. `distinctUntilChanged` prevents unnecessary API calls when the value hasn't actually changed, improving performance.

## Key Takeaways

1. **filter** - Emit only values that pass a test
2. **debounceTime** - Wait for pause in emissions (search inputs)
3. **distinctUntilChanged** - Emit only when value changes
4. **take** - Emit first n values, then complete
5. **takeUntil** - Emit until notifier observable emits (cleanup pattern)
6. **throttleTime** - Emit first value, then limit rate (button clicks)
7. Always unsubscribe using **takeUntilDestroyed** or **takeUntil**
8. Combine multiple filtering operators for optimal control
9. Use type guards with filter for type safety
10. Choose the right timing operator for your use case

## Resources

- [RxJS Documentation - Filtering Operators](https://rxjs.dev/guide/operators#filtering-operators)
- [Learn RxJS - Filtering](https://www.learnrxjs.io/learn-rxjs/operators/filtering)
- [Angular University - RxJS Filtering](https://blog.angular-university.io/rxjs-filtering-operators/)
- [RxJS Marbles](https://rxmarbles.com/)
