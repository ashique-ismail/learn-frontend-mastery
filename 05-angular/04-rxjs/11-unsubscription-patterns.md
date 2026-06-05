# RxJS Unsubscription Patterns

## The Idea

**In plain English:** When your app "listens" to a stream of data (like a live news feed), it needs to stop listening when it no longer needs the data — otherwise the app keeps wasting memory running a listener nobody is using. Unsubscription patterns are the different ways you tell your app to stop listening at the right time.

**Real-world analogy:** Imagine you hire a newspaper delivery service while you live in an apartment. When you move out, you must cancel the subscription — otherwise newspapers pile up at your old door forever, and the delivery person keeps working for nothing. Different ways of canceling (calling ahead, setting an end date, auto-canceling when you hand in your keys) are the unsubscription patterns.

- The newspaper delivery = the observable (the ongoing stream of data)
- You reading the newspaper = the subscription (your code consuming the data)
- Moving out of the apartment = the component being destroyed
- Canceling the delivery = unsubscribing

---

## Table of Contents

- [Introduction](#introduction)
- [Why Unsubscription Matters](#why-unsubscription-matters)
- [takeUntilDestroyed](#takeuntildestroyed)
- [async Pipe](#async-pipe)
- [takeUntil Pattern](#takeuntil-pattern)
- [Manual Unsubscribe](#manual-unsubscribe)
- [take and first](#take-and-first)
- [Subscription Management](#subscription-management)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Memory leaks are one of the most common issues in Angular applications, primarily caused by forgotten subscriptions. When a component is destroyed but its subscriptions remain active, they continue to consume memory and execute callbacks, leading to performance degradation and unexpected behavior.

Understanding unsubscription patterns is critical for building reliable, performant Angular applications. This guide covers modern and traditional approaches to managing observable subscriptions.

## Why Unsubscription Matters

### Memory Leak Example

```typescript
@Component({
  selector: 'app-leaky'
})
export class LeakyComponent implements OnInit {
  private dataService = inject(DataService);

  ngOnInit() {
    // PROBLEM: This subscription never cleans up
    interval(1000).subscribe(tick => {
      console.log('Tick:', tick);
      this.updateUI();
    });

    // PROBLEM: This HTTP subscription might complete, but it's still bad practice
    this.dataService.getData().subscribe(data => {
      console.log('Data:', data);
    });
  }

  private updateUI() {
    // Component might be destroyed, but this still runs!
    // Accessing component properties can cause errors
  }
}

// Every time this component is created and destroyed,
// a new timer keeps running forever!
```

### Symptoms of Memory Leaks

1. **Increasing memory usage** over time
2. **Duplicate subscriptions** executing
3. **Errors after navigation** (accessing destroyed components)
4. **Performance degradation** (too many active subscriptions)
5. **Unexpected behavior** (stale callbacks executing)

### Which Observables Need Unsubscription?

```typescript
// NEED unsubscription (long-lived):
interval(1000)           // Runs forever
fromEvent(el, 'click')   // Listens forever
subject.asObservable()   // Depends on subject lifecycle
webSocket('ws://...')    // Connection stays open

// DON'T need unsubscription (auto-complete):
http.get('/api/data')    // Completes after response
of(1, 2, 3)             // Completes immediately
from([1, 2, 3])         // Completes immediately
timer(1000)             // Completes after one emission
```

## takeUntilDestroyed

The modern, recommended approach for Angular 16+. Automatically unsubscribes when the component is destroyed.

### Basic Usage (Angular 16+)

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-modern'
})
export class ModernComponent implements OnInit {
  private dataService = inject(DataService);

  ngOnInit() {
    // Automatically unsubscribes when component is destroyed
    interval(1000).pipe(
      takeUntilDestroyed()
    ).subscribe(tick => {
      console.log('Tick:', tick);
    });
  }
}
```

### With DestroyRef (Explicit)

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef } from '@angular/core';

@Component({
  selector: 'app-explicit-destroy'
})
export class ExplicitDestroyComponent implements OnInit {
  private destroyRef = inject(DestroyRef);
  private dataService = inject(DataService);

  ngOnInit() {
    // Pass DestroyRef explicitly
    this.dataService.getData().pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(data => {
      console.log('Data:', data);
    });
  }
}
```

### In Constructor (No DestroyRef Needed)

```typescript
@Component({
  selector: 'app-constructor-pattern'
})
export class ConstructorPatternComponent {
  private dataService = inject(DataService);

  // In constructor, takeUntilDestroyed uses injection context automatically
  data$ = this.dataService.getData().pipe(
    takeUntilDestroyed()
  );

  constructor() {
    // Can also subscribe directly in constructor
    interval(1000).pipe(
      takeUntilDestroyed()
    ).subscribe(tick => {
      console.log('Tick:', tick);
    });
  }
}
```

### With Multiple Observables

```typescript
@Component({
  selector: 'app-multiple-streams'
})
export class MultipleStreamsComponent implements OnInit {
  private service1 = inject(Service1);
  private service2 = inject(Service2);
  private service3 = inject(Service3);

  ngOnInit() {
    // All automatically clean up
    this.service1.data$.pipe(
      takeUntilDestroyed()
    ).subscribe(data => this.handleData1(data));

    this.service2.data$.pipe(
      takeUntilDestroyed()
    ).subscribe(data => this.handleData2(data));

    this.service3.data$.pipe(
      takeUntilDestroyed()
    ).subscribe(data => this.handleData3(data));
  }
}
```

### Real-World Example

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="metrics">
      <div>Active Users: {{ activeUsers }}</div>
      <div>Server Status: {{ serverStatus }}</div>
      <div>Requests/sec: {{ requestsPerSecond }}</div>
    </div>
  `
})
export class DashboardComponent implements OnInit {
  private metricsService = inject(MetricsService);

  activeUsers = 0;
  serverStatus = 'Unknown';
  requestsPerSecond = 0;

  ngOnInit() {
    // Poll active users every 5 seconds
    interval(5000).pipe(
      takeUntilDestroyed(),
      switchMap(() => this.metricsService.getActiveUsers())
    ).subscribe(count => {
      this.activeUsers = count;
    });

    // WebSocket for real-time status
    this.metricsService.serverStatus$.pipe(
      takeUntilDestroyed()
    ).subscribe(status => {
      this.serverStatus = status;
    });

    // Compute requests per second from event stream
    this.metricsService.requestEvents$.pipe(
      takeUntilDestroyed(),
      bufferTime(1000),
      map(events => events.length)
    ).subscribe(count => {
      this.requestsPerSecond = count;
    });
  }
}
```

## async Pipe

The async pipe automatically subscribes AND unsubscribes, making it the safest pattern for templates.

### Basic Usage

```typescript
@Component({
  selector: 'app-async-pattern',
  template: `
    <div *ngIf="user$ | async as user">
      {{ user.name }}
    </div>
  `
})
export class AsyncPatternComponent {
  private userService = inject(UserService);

  // No manual subscription needed!
  user$ = this.userService.getCurrentUser();
}
```

### Multiple Async Pipes (Same Observable)

```typescript
@Component({
  selector: 'app-multiple-async',
  template: `
    <!-- PROBLEM: Multiple subscriptions to same observable -->
    <div>Name: {{ user$ | async }}</div>
    <div>Email: {{ user$ | async }}</div>
    <div>Role: {{ user$ | async }}</div>
    <!-- Creates 3 subscriptions! -->
  `
})
export class MultipleAsyncComponent {
  user$ = this.userService.getUser();
}

// SOLUTION 1: Use structural directive
@Component({
  selector: 'app-single-async',
  template: `
    <div *ngIf="user$ | async as user">
      <div>Name: {{ user.name }}</div>
      <div>Email: {{ user.email }}</div>
      <div>Role: {{ user.role }}</div>
    </div>
  `
})
export class SingleAsyncComponent {
  user$ = this.userService.getUser();
}

// SOLUTION 2: Use shareReplay
@Component({
  selector: 'app-shared-async'
})
export class SharedAsyncComponent {
  user$ = this.userService.getUser().pipe(
    shareReplay(1) // Share subscription
  );
}
```

### Async Pipe with Loading States

```typescript
@Component({
  selector: 'app-with-loading',
  template: `
    <div *ngIf="data$ | async as data; else loading">
      <div *ngFor="let item of data">{{ item.name }}</div>
    </div>
    <ng-template #loading>
      <div>Loading...</div>
    </ng-template>
  `
})
export class WithLoadingComponent {
  private dataService = inject(DataService);

  data$ = this.dataService.getData().pipe(
    startWith(null as any) // Show loading initially
  );
}
```

### Combining Multiple Observables

```typescript
@Component({
  selector: 'app-combined',
  template: `
    <div *ngIf="vm$ | async as vm">
      <h1>{{ vm.user.name }}</h1>
      <div>{{ vm.products.length }} products</div>
      <div>{{ vm.notifications.length }} notifications</div>
    </div>
  `
})
export class CombinedComponent {
  private userService = inject(UserService);
  private productService = inject(ProductService);
  private notificationService = inject(NotificationService);

  // Single subscription in template
  vm$ = combineLatest({
    user: this.userService.getCurrentUser(),
    products: this.productService.getProducts(),
    notifications: this.notificationService.getNotifications()
  });
}
```

### Error Handling with Async Pipe

```typescript
@Component({
  selector: 'app-with-error-handling',
  template: `
    <div *ngIf="data$ | async as data; else errorTemplate">
      {{ data }}
    </div>
    <ng-template #errorTemplate>
      <div class="error">Failed to load data</div>
    </ng-template>
  `
})
export class WithErrorHandlingComponent {
  private dataService = inject(DataService);

  data$ = this.dataService.getData().pipe(
    catchError(error => {
      console.error('Error loading data:', error);
      return of(null); // Return null to show error template
    })
  );
}
```

## takeUntil Pattern

The traditional pattern using a Subject that emits when the component is destroyed. Still useful for Angular 15 and below.

### Basic Pattern

```typescript
import { Subject, takeUntil } from 'rxjs';

@Component({
  selector: 'app-take-until'
})
export class TakeUntilComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private dataService = inject(DataService);

  ngOnInit() {
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      console.log('Data:', data);
    });

    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(tick => {
      console.log('Tick:', tick);
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Reusable Base Class

```typescript
import { Directive, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';

@Directive()
export abstract class DestroyableComponent implements OnDestroy {
  protected destroy$ = new Subject<void>();

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Usage
@Component({
  selector: 'app-child'
})
export class ChildComponent extends DestroyableComponent implements OnInit {
  private dataService = inject(DataService);

  ngOnInit() {
    // this.destroy$ available from base class
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      console.log('Data:', data);
    });
  }
}
```

### takeUntil with Operators

```typescript
@Component({
  selector: 'app-complex-stream'
})
export class ComplexStreamComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private searchService = inject(SearchService);

  searchControl = new FormControl('');

  ngOnInit() {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(term => term.length >= 3),
      switchMap(term => this.searchService.search(term)),
      takeUntil(this.destroy$) // Place at end of chain
    ).subscribe(results => {
      console.log('Search results:', results);
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Manual Unsubscribe

Directly managing Subscription objects. Most verbose but gives complete control.

### Single Subscription

```typescript
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-manual'
})
export class ManualComponent implements OnInit, OnDestroy {
  private subscription: Subscription | undefined;
  private dataService = inject(DataService);

  ngOnInit() {
    this.subscription = this.dataService.getData().subscribe(data => {
      console.log('Data:', data);
    });
  }

  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}
```

### Multiple Subscriptions (Array)

```typescript
@Component({
  selector: 'app-multiple-subs'
})
export class MultipleSubsComponent implements OnInit, OnDestroy {
  private subscriptions: Subscription[] = [];

  ngOnInit() {
    this.subscriptions.push(
      interval(1000).subscribe(tick => console.log('Tick:', tick))
    );

    this.subscriptions.push(
      fromEvent(window, 'resize').subscribe(() => console.log('Resized'))
    );

    this.subscriptions.push(
      this.dataService.getData().subscribe(data => console.log('Data:', data))
    );
  }

  ngOnDestroy() {
    this.subscriptions.forEach(sub => sub.unsubscribe());
  }
}
```

### Subscription Container

```typescript
@Component({
  selector: 'app-container'
})
export class ContainerComponent implements OnInit, OnDestroy {
  private subscription = new Subscription();

  ngOnInit() {
    // Add all subscriptions to container
    this.subscription.add(
      interval(1000).subscribe(tick => console.log('Tick:', tick))
    );

    this.subscription.add(
      this.dataService.getData().subscribe(data => console.log('Data:', data))
    );

    this.subscription.add(
      fromEvent(window, 'scroll').subscribe(() => console.log('Scrolled'))
    );
  }

  ngOnDestroy() {
    // Unsubscribe all at once
    this.subscription.unsubscribe();
  }
}
```

### Conditional Unsubscription

```typescript
@Component({
  selector: 'app-conditional'
})
export class ConditionalComponent implements OnInit {
  private pollingSubscription: Subscription | undefined;

  startPolling() {
    // Stop existing polling if any
    this.stopPolling();

    this.pollingSubscription = interval(5000).pipe(
      switchMap(() => this.dataService.getData())
    ).subscribe(data => {
      console.log('Polled data:', data);
    });
  }

  stopPolling() {
    this.pollingSubscription?.unsubscribe();
    this.pollingSubscription = undefined;
  }

  ngOnDestroy() {
    this.stopPolling();
  }
}
```

## take and first

Operators that automatically complete after a certain number of emissions.

### take Operator

```typescript
@Component({
  selector: 'app-take-example'
})
export class TakeExampleComponent implements OnInit {
  ngOnInit() {
    // Take only first 5 values, then complete
    interval(1000).pipe(
      take(5)
    ).subscribe({
      next: tick => console.log('Tick:', tick),
      complete: () => console.log('Completed after 5 ticks')
    });
    // Automatically unsubscribes after 5 emissions
  }
}
```

### first Operator

```typescript
@Component({
  selector: 'app-first-example'
})
export class FirstExampleComponent implements OnInit {
  private userService = inject(UserService);

  ngOnInit() {
    // Take first emission, then complete
    this.userService.users$.pipe(
      first()
    ).subscribe(users => {
      console.log('Initial users:', users);
    });
    // Automatically unsubscribes after first emission

    // First matching condition
    this.userService.users$.pipe(
      first(users => users.length > 10)
    ).subscribe(users => {
      console.log('Found 10+ users:', users);
    });
  }
}
```

### take(1) vs first() vs single()

```typescript
@Component({
  selector: 'app-comparison'
})
export class ComparisonComponent {
  // take(1) - Take first value, complete
  takeSub() {
    source$.pipe(take(1)).subscribe(
      value => console.log(value)
    );
  }

  // first() - Take first value (with optional predicate), error if none
  firstSub() {
    source$.pipe(first()).subscribe(
      value => console.log(value),
      error => console.error('No values emitted')
    );
  }

  // first() with predicate
  firstWithPredicate() {
    source$.pipe(
      first(value => value > 10)
    ).subscribe(
      value => console.log(value),
      error => console.error('No matching value')
    );
  }
}
```

## Subscription Management

### Service with Cleanup

```typescript
@Injectable()
export class ManagedService implements OnDestroy {
  private subscription = new Subscription();

  constructor(private http: HttpClient) {
    // Service-level subscriptions
    this.subscription.add(
      interval(60000).subscribe(() => this.refreshData())
    );
  }

  private refreshData() {
    this.http.get('/api/data').subscribe(/* ... */);
  }

  ngOnDestroy() {
    this.subscription.unsubscribe();
  }
}
```

### Component with Service Cleanup

```typescript
@Component({
  selector: 'app-managed'
})
export class ManagedComponent implements OnInit {
  private userService = inject(UserService);
  private destroy$ = new Subject<void>();

  // Declare as observables, let template handle with async pipe
  users$ = this.userService.getUsers();
  
  // Or manually manage if needed
  ngOnInit() {
    this.userService.notifications$.pipe(
      takeUntil(this.destroy$),
      tap(notification => this.showToast(notification))
    ).subscribe();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Common Mistakes

### 1. Forgetting to Unsubscribe

```typescript
// WRONG: Memory leak
ngOnInit() {
  interval(1000).subscribe(tick => {
    console.log('Tick:', tick);
  });
}

// RIGHT: Clean up
ngOnInit() {
  interval(1000).pipe(
    takeUntilDestroyed()
  ).subscribe(tick => {
    console.log('Tick:', tick);
  });
}
```

### 2. Unsubscribing from HTTP

```typescript
// UNNECESSARY: HTTP requests auto-complete
ngOnInit() {
  this.subscription = this.http.get('/api/data').subscribe();
}

// BETTER: No manual cleanup needed (but doesn't hurt)
ngOnInit() {
  this.http.get('/api/data').subscribe();
}
```

### 3. Multiple Async Pipes on Same Observable

```typescript
// WRONG: Multiple subscriptions
<div>{{ data$ | async }}</div>
<div>{{ data$ | async }}</div>

// RIGHT: Share or use structural directive
<div *ngIf="data$ | async as data">
  <div>{{ data }}</div>
  <div>{{ data }}</div>
</div>
```

### 4. Not Completing Subject

```typescript
// WRONG: Subject not completed
ngOnDestroy() {
  this.destroy$.next();
  // Missing: this.destroy$.complete();
}

// RIGHT: Always complete
ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### 5. takeUntil in Wrong Position

```typescript
// WRONG: takeUntil won't catch inner observable
source$.pipe(
  takeUntil(this.destroy$),
  switchMap(() => innerObservable$) // This can leak!
)

// RIGHT: takeUntil after all operators
source$.pipe(
  switchMap(() => innerObservable$),
  takeUntil(this.destroy$) // Catches everything
)
```

## Best Practices

### 1. Prefer Declarative (Async Pipe)

```typescript
// BEST: No manual subscription management
@Component({
  template: `<div>{{ data$ | async }}</div>`
})
export class BestComponent {
  data$ = this.service.getData();
}
```

### 2. Use takeUntilDestroyed (Angular 16+)

```typescript
// MODERN: Automatic cleanup
ngOnInit() {
  this.service.data$.pipe(
    takeUntilDestroyed()
  ).subscribe();
}
```

### 3. Use takeUntil for Older Versions

```typescript
// TRADITIONAL: Manual but reliable
private destroy$ = new Subject<void>();

ngOnInit() {
  this.service.data$.pipe(
    takeUntil(this.destroy$)
  ).subscribe();
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### 4. One takeUntil at End of Chain

```typescript
// GOOD: Single takeUntil at end
source$.pipe(
  map(x => x * 2),
  filter(x => x > 10),
  switchMap(x => this.service.get(x)),
  takeUntilDestroyed() // At the end
).subscribe();
```

### 5. Document Subscription Patterns

```typescript
/**
 * Component that manages real-time data subscriptions.
 * Uses takeUntilDestroyed() for automatic cleanup.
 */
@Component({
  selector: 'app-documented'
})
export class DocumentedComponent {
  // ...
}
```

## Interview Questions

### Q1: Why is unsubscription important?

**Answer:** Unsubscription prevents memory leaks by cleaning up listeners, timers, and callbacks when components are destroyed. Without it, subscriptions continue running, consuming memory and potentially causing errors when trying to access destroyed component properties.

### Q2: Which observables don't need unsubscription?

**Answer:** Observables that complete automatically:

- HTTP requests (complete after response)
- `of()`, `from()` with finite arrays
- `timer()` with one emission
- Observables with `take()`, `first()`, `takeWhile()` that guarantee completion

Long-lived observables always need cleanup: `interval()`, `fromEvent()`, Subjects, WebSockets.

### Q3: What's the difference between takeUntil and takeWhile?

**Answer:**

- `takeUntil(notifier$)`: Completes when notifier emits (perfect for cleanup on destroy)
- `takeWhile(predicate)`: Completes when predicate returns false (condition-based)

Use `takeUntil` for component cleanup, `takeWhile` for conditional streaming.

### Q4: Why should takeUntil be at the end of the pipe?

**Answer:** Operators like `switchMap` and `mergeMap` create inner subscriptions. Placing `takeUntil` at the end ensures it unsubscribes from both outer and inner observables. If placed earlier, inner observables might leak.

## Key Takeaways

1. Always clean up long-lived subscriptions
2. **Prefer async pipe** - automatic subscription management
3. **Use takeUntilDestroyed()** in Angular 16+ for imperative subscriptions
4. **Use takeUntil pattern** for Angular 15 and below
5. HTTP requests auto-complete but cleanup doesn't hurt
6. Place `takeUntil` at the **end** of operator chains
7. Complete Subjects in `ngOnDestroy`
8. Avoid multiple async pipes on same observable
9. Use `shareReplay()` when multiple subscriptions needed
10. Memory leaks cause performance issues and bugs

## Resources

- [Angular RxJS Interop Documentation](https://angular.io/api/core/rxjs-interop/takeUntilDestroyed)
- [RxJS Documentation - takeUntil](https://rxjs.dev/api/operators/takeUntil)
- [Angular University - Unsubscription](https://blog.angular-university.io/rxjs-unsubscribe/)
- [Ben Lesh - RxJS: Don't Unsubscribe](https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87)
