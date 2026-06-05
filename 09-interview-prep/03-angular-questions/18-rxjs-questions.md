# Angular RxJS Interview Questions

## The Idea

**In plain English:** RxJS is a toolkit for handling things that happen over time in your app — like a user typing, a button being clicked, or data arriving from a server. Instead of waiting around or checking repeatedly, you set up a "watcher" that reacts automatically whenever something new happens.

**Real-world analogy:** Think of a newspaper subscription. A printing press keeps producing new editions, and subscribers receive each new issue as it arrives.

- The printing press = the Observable (the source that produces values over time)
- The newspaper edition = the emitted value (each piece of data sent down the stream)
- Your subscription = the Subscriber (the code that runs each time a new value arrives)

---

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

RxJS (Reactive Extensions for JavaScript) is fundamental to Angular development. It provides powerful tools for handling asynchronous operations, events, and data streams using observables.

### Observable Basics

**Key Concepts:**
- Observables represent streams of values over time
- Observers subscribe to observables
- Operators transform observable streams
- Subscriptions can be cancelled
- Hot vs Cold observables

### Common Operators

**Creation**: of, from, interval, timer, fromEvent
**Transformation**: map, mergeMap, switchMap, concatMap, exhaustMap
**Filtering**: filter, take, skip, distinctUntilChanged, debounceTime
**Combination**: combineLatest, merge, concat, forkJoin, zip
**Error Handling**: catchError, retry, retryWhen
**Utility**: tap, delay, finalize

## Common Interview Questions

### Q1: Explain the difference between map, mergeMap, switchMap, concatMap, and exhaustMap.

**What the interviewer wants to know:**
- Understanding of flattening operators
- Knowledge of when to use each operator
- Awareness of common async patterns

**Strong Answer:**

These are all flattening operators that handle nested observables, but they differ in how they manage subscriptions and emit values.

**map - Simple Transformation:**

```typescript
// map: Transform each value (NOT for observables)
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

// Simple value transformation
of(1, 2, 3).pipe(
  map(x => x * 2)
).subscribe(console.log);
// Output: 2, 4, 6

// ❌ WRONG: Don't use map for observables
this.userService.getUser(1).pipe(
  map(user => this.orderService.getOrders(user.id))
).subscribe(orders$ => {
  // orders$ is an Observable, not orders!
  console.log(orders$); // Observable { ... }
});

// ✅ CORRECT: Use flatMap operators for observables
this.userService.getUser(1).pipe(
  mergeMap(user => this.orderService.getOrders(user.id))
).subscribe(orders => {
  console.log(orders); // Actual orders array
});
```

**mergeMap (flatMap) - Concurrent Execution:**

```typescript
// mergeMap: Subscribes to all inner observables concurrently
// Maintains all subscriptions, emits values as they arrive
import { of, interval } from 'rxjs';
import { mergeMap, take } from 'rxjs/operators';

// Example: Concurrent API calls
of(1, 2, 3).pipe(
  mergeMap(id => 
    this.http.get(`/api/user/${id}`)
  )
).subscribe(user => console.log(user));
// Users arrive in whatever order API responds
// All 3 requests happen simultaneously

// Real-world: Search with autocomplete
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  mergeMap(query => this.searchService.search(query))
).subscribe(results => {
  // Problem: If user types quickly, old search results
  // can arrive after new ones, showing wrong data
});
```

**switchMap - Cancel Previous:**

```typescript
// switchMap: Cancels previous inner observable when new value arrives
// Only the latest subscription is active
import { fromEvent } from 'rxjs';
import { switchMap, debounceTime } from 'rxjs/operators';

// Example: Search that cancels previous requests
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  switchMap(query => this.searchService.search(query))
).subscribe(results => {
  // Only shows results from the latest query
  // Previous search requests are cancelled
  this.searchResults = results;
});

// Real-world: Route parameters
this.route.params.pipe(
  switchMap(params => this.userService.getUser(params['id']))
).subscribe(user => {
  // If user navigates to different user before request completes,
  // previous request is cancelled
  this.user = user;
});

// Visual timeline:
// User types: 'a' -> 'an' -> 'ang' -> 'angu' -> 'angular'
//             |      |       |        |          |
// Requests:   X      X       X        X          ✓ (only this completes)
// switchMap cancels all previous requests
```

**concatMap - Sequential Execution:**

```typescript
// concatMap: Waits for previous inner observable to complete
// Maintains order, processes one at a time
import { of } from 'rxjs';
import { concatMap, delay } from 'rxjs/operators';

// Example: Sequential API calls
of(1, 2, 3).pipe(
  concatMap(id => 
    this.http.post('/api/process', { id }).pipe(
      delay(1000) // Simulate slow request
    )
  )
).subscribe(result => console.log(result));
// Processes: 1... waits... 2... waits... 3...
// Total time: ~3 seconds

// Real-world: Order matters (queue processing)
this.uploadQueue$.pipe(
  concatMap(file => this.uploadService.upload(file))
).subscribe(
  result => console.log(`Uploaded: ${result.filename}`),
  error => console.error(`Upload failed`)
);
// Files upload one at a time, in order
```

**exhaustMap - Ignore While Busy:**

```typescript
// exhaustMap: Ignores new values while current inner observable is active
// Prevents multiple simultaneous subscriptions
import { fromEvent } from 'rxjs';
import { exhaustMap } from 'rxjs/operators';

// Example: Prevent double-click submission
this.submitButton.clicks$.pipe(
  exhaustMap(() => this.formService.submit(this.formData))
).subscribe(
  result => console.log('Form submitted successfully'),
  error => console.error('Submission failed')
);
// If user clicks multiple times quickly, only first click processes
// Additional clicks are ignored until submission completes

// Real-world: Login button
fromEvent(loginButton, 'click').pipe(
  exhaustMap(() => 
    this.authService.login(this.credentials).pipe(
      catchError(error => {
        this.showError(error);
        return EMPTY;
      })
    )
  )
).subscribe(user => {
  // User can't trigger multiple login attempts
  // while one is in progress
  this.router.navigate(['/dashboard']);
});

// Visual timeline:
// Button clicks: 1  2  3  4        5
//                |  X  X  X        |
// Requests:      |................ |..............
//                ✓ (2,3,4 ignored) ✓
```

**Comparison Table:**

| Operator | Use When | Cancels Previous? | Maintains Order? | Common Use Case |
|----------|----------|-------------------|------------------|-----------------|
| **map** | Transforming values (not observables) | N/A | N/A | Data transformation |
| **mergeMap** | Concurrent execution acceptable | No | No | Parallel API calls |
| **switchMap** | Want latest value only | Yes | No | Search, navigation |
| **concatMap** | Order matters, sequential | Waits | Yes | Upload queue |
| **exhaustMap** | Prevent overlapping operations | Ignores new | N/A | Form submission |

**Practical Decision Tree:**

```typescript
// Decision helper
class OperatorSelector {
  chooseOperator(scenario: string): string {
    // Does the function return an Observable?
    if (!returnsObservable) {
      return 'Use map';
    }
    
    // Do you only care about the latest result?
    if (onlyLatestMatters) {
      return 'Use switchMap'; // Search, typeahead
    }
    
    // Must operations complete in order?
    if (orderMatters) {
      return 'Use concatMap'; // File uploads, transactions
    }
    
    // Should you ignore new values while processing?
    if (preventOverlap) {
      return 'Use exhaustMap'; // Form submission, login
    }
    
    // Want concurrent execution?
    return 'Use mergeMap'; // Independent parallel operations
  }
}
```

**Real-World Example Combining Multiple:**

```typescript
@Component({
  selector: 'app-user-dashboard',
  template: `
    <input [formControl]="searchControl" placeholder="Search users">
    <button (click)="refresh()">Refresh</button>
    <div *ngFor="let user of users">
      {{ user.name }}
      <button (click)="loadOrders(user.id)">Load Orders</button>
    </div>
  `
})
export class UserDashboardComponent implements OnInit {
  searchControl = new FormControl('');
  users: User[] = [];
  private refreshSubject = new Subject<void>();

  ngOnInit(): void {
    // switchMap for search (cancel previous searches)
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(query => 
        query ? this.userService.search(query) : of([])
      )
    ).subscribe(users => {
      this.users = users;
    });

    // exhaustMap for refresh (prevent multiple simultaneous refreshes)
    this.refreshSubject.pipe(
      exhaustMap(() => this.userService.getAll())
    ).subscribe(users => {
      this.users = users;
    });
  }

  refresh(): void {
    this.refreshSubject.next();
  }

  loadOrders(userId: number): void {
    // mergeMap for concurrent order loading
    // (multiple users can load orders simultaneously)
    of(userId).pipe(
      mergeMap(id => this.orderService.getOrders(id))
    ).subscribe(orders => {
      console.log(`Orders for user ${userId}:`, orders);
    });
  }
}
```

**Follow-up Questions:**

1. **What's the difference between mergeMap and forkJoin?**
   - mergeMap: Flattens observables as they arrive, emits multiple times
   - forkJoin: Waits for all observables to complete, emits once with array
   - mergeMap for continuous streams, forkJoin for parallel batch operations

2. **When would concatMap cause performance issues?**
   - Long-running operations queue up
   - Creates backpressure if new values arrive faster than processing
   - Can cause memory issues with large queues
   - Better to use switchMap or debounce if order doesn't matter

3. **How do you cancel an exhaustMap operation?**
   - You can't manually cancel it
   - It automatically ignores new values while busy
   - To force cancellation, use switchMap instead
   - Or unsubscribe from the outer observable

---

### Q2: Explain hot vs cold observables and provide examples of each.

**What the interviewer wants to know:**
- Understanding of observable behavior
- Knowledge of when subscriptions trigger data production
- Awareness of common pitfalls

**Strong Answer:**

The distinction between hot and cold observables is crucial for understanding data flow and avoiding bugs in Angular applications.

**Cold Observables - Unicast:**

```typescript
// Cold: Each subscription creates a new execution
// Data production starts when you subscribe
import { Observable } from 'rxjs';

// Example: HTTP request (cold)
const user$ = this.http.get<User>('/api/user/1');

user$.subscribe(user => console.log('Sub 1:', user));
// HTTP request #1 sent

user$.subscribe(user => console.log('Sub 2:', user));
// HTTP request #2 sent (separate request!)

// Each subscriber gets its own independent data stream
```

**Cold Observable Characteristics:**

```typescript
// 1. Timer/Interval (cold)
const timer$ = interval(1000);

timer$.subscribe(n => console.log('Sub A:', n));
// Sub A: 0, 1, 2, 3...

setTimeout(() => {
  timer$.subscribe(n => console.log('Sub B:', n));
  // Sub B starts from 0, not from where Sub A is
  // Sub B: 0, 1, 2, 3...
}, 3000);

// 2. Observable.create (cold)
const cold$ = new Observable(observer => {
  console.log('Observable execution started');
  observer.next(Math.random());
  observer.complete();
});

cold$.subscribe(val => console.log('Sub 1:', val));
// Output: "Observable execution started"
//         "Sub 1: 0.123"

cold$.subscribe(val => console.log('Sub 2:', val));
// Output: "Observable execution started"
//         "Sub 2: 0.456"
// Each subscription triggers new execution!
```

**Hot Observables - Multicast:**

```typescript
// Hot: Data production is independent of subscriptions
// Subscribers share the same execution
import { Subject } from 'rxjs';

// Example: Subject (hot)
const clicks$ = new Subject<MouseEvent>();

clicks$.subscribe(event => console.log('Sub 1:', event.clientX));
clicks$.subscribe(event => console.log('Sub 2:', event.clientX));

document.addEventListener('click', event => {
  clicks$.next(event);
});
// Both subscribers receive the same click events
// Single data source, multiple consumers
```

**Hot Observable Characteristics:**

```typescript
// 1. Subjects (hot)
const subject$ = new Subject<number>();

subject$.subscribe(n => console.log('Sub A:', n));

subject$.next(1); // Sub A: 1
subject$.next(2); // Sub A: 2

subject$.subscribe(n => console.log('Sub B:', n));
// Sub B doesn't receive 1 and 2 (already emitted)

subject$.next(3); // Sub A: 3, Sub B: 3

// 2. DOM Events (hot)
const clicks$ = fromEvent(document, 'click');

clicks$.subscribe(e => console.log('Sub 1 clicked'));
// Doesn't create new click events
// Just taps into existing event stream

setTimeout(() => {
  clicks$.subscribe(e => console.log('Sub 2 clicked'));
  // Sub 2 won't receive past clicks, only future ones
}, 5000);

// 3. WebSocket (hot)
const socket$ = webSocket('ws://localhost:8080');

socket$.subscribe(msg => console.log('Sub 1:', msg));
socket$.subscribe(msg => console.log('Sub 2:', msg));
// Both receive same WebSocket messages
// Single WebSocket connection shared
```

**Converting Cold to Hot:**

```typescript
// Using share() operator
import { share } from 'rxjs/operators';

// Cold observable
const coldHttp$ = this.http.get<User>('/api/user/1');

coldHttp$.subscribe(user => console.log('Sub 1:', user)); // HTTP request #1
coldHttp$.subscribe(user => console.log('Sub 2:', user)); // HTTP request #2

// Convert to hot
const hotHttp$ = this.http.get<User>('/api/user/1').pipe(
  share() // Multicasts to multiple subscribers
);

hotHttp$.subscribe(user => console.log('Sub 1:', user)); // HTTP request
hotHttp$.subscribe(user => console.log('Sub 2:', user)); // No new request!
// Single HTTP request, both subscribers receive result

// Using shareReplay() - Caching
import { shareReplay } from 'rxjs/operators';

const cachedUser$ = this.http.get<User>('/api/user/1').pipe(
  shareReplay(1) // Cache last 1 value
);

cachedUser$.subscribe(user => console.log('Sub 1:', user)); // HTTP request

setTimeout(() => {
  cachedUser$.subscribe(user => console.log('Sub 2:', user)); // Gets cached value
}, 5000);
```

**Real-World Patterns:**

**1. Caching HTTP Requests:**

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private userCache$ = new Map<number, Observable<User>>();

  getUser(id: number): Observable<User> {
    if (!this.userCache$.has(id)) {
      // Create hot observable with caching
      this.userCache$.set(
        id,
        this.http.get<User>(`/api/users/${id}`).pipe(
          shareReplay(1) // Cache the result
        )
      );
    }
    return this.userCache$.get(id)!;
  }

  clearCache(id?: number): void {
    if (id) {
      this.userCache$.delete(id);
    } else {
      this.userCache$.clear();
    }
  }
}

// Usage
this.userService.getUser(1).subscribe(user => console.log(user));
// HTTP request sent

this.userService.getUser(1).subscribe(user => console.log(user));
// Uses cached value, no HTTP request
```

**2. WebSocket Stream:**

```typescript
@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private socket$: Observable<any>;

  connect(): Observable<any> {
    if (!this.socket$) {
      // Create hot observable for WebSocket
      this.socket$ = new Observable(observer => {
        const ws = new WebSocket('ws://localhost:8080');
        
        ws.onmessage = event => observer.next(event.data);
        ws.onerror = error => observer.error(error);
        ws.onclose = () => observer.complete();
        
        return () => ws.close();
      }).pipe(
        share() // Multiple components share same WebSocket
      );
    }
    
    return this.socket$;
  }
}

// Multiple components subscribe
@Component({})
export class ChatComponent {
  constructor(private ws: WebSocketService) {
    this.ws.connect().subscribe(msg => {
      // All components receive same messages
      this.messages.push(msg);
    });
  }
}
```

**3. Event Bus:**

```typescript
@Injectable({ providedIn: 'root' })
export class EventBus {
  // Hot observable - all subscribers receive events
  private eventSubject = new Subject<Event>();
  public events$ = this.eventSubject.asObservable();

  emit(event: Event): void {
    this.eventSubject.next(event);
  }
}

// Component A
export class ComponentA {
  constructor(private eventBus: EventBus) {}
  
  doSomething(): void {
    this.eventBus.emit({ type: 'USER_ACTION', data: {} });
  }
}

// Component B
export class ComponentB {
  constructor(private eventBus: EventBus) {
    // Receives events from any component
    this.eventBus.events$.subscribe(event => {
      console.log('Received event:', event);
    });
  }
}
```

**Common Pitfalls:**

```typescript
// ❌ PITFALL 1: Assuming HTTP is hot
const users$ = this.http.get('/api/users');

users$.subscribe(); // Request #1
users$.subscribe(); // Request #2 (unexpected!)

// ✅ FIX: Use share() or shareReplay()
const users$ = this.http.get('/api/users').pipe(
  shareReplay(1)
);

// ❌ PITFALL 2: Memory leaks with hot observables
@Component({})
export class MyComponent implements OnInit {
  ngOnInit(): void {
    this.eventBus.events$.subscribe(event => {
      // Subscription never cleaned up!
    });
  }
}

// ✅ FIX: Unsubscribe or use async pipe
export class MyComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    this.eventBus.events$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(event => {});
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ❌ PITFALL 3: Late subscribers miss values
const subject$ = new Subject<number>();

subject$.next(1);
subject$.next(2);

subject$.subscribe(n => console.log(n));
// Won't receive 1 and 2!

// ✅ FIX: Use BehaviorSubject or ReplaySubject
const behaviorSubject$ = new BehaviorSubject<number>(0);

behaviorSubject$.next(1);
behaviorSubject$.next(2);

behaviorSubject$.subscribe(n => console.log(n));
// Receives: 2 (latest value)

const replaySubject$ = new ReplaySubject<number>(2);

replaySubject$.next(1);
replaySubject$.next(2);

replaySubject$.subscribe(n => console.log(n));
// Receives: 1, 2 (last 2 values)
```

**Testing Hot vs Cold:**

```typescript
describe('Observable Types', () => {
  it('should demonstrate cold observable', () => {
    const values: number[] = [];
    const cold$ = of(1, 2, 3);
    
    cold$.subscribe(n => values.push(n));
    cold$.subscribe(n => values.push(n));
    
    expect(values).toEqual([1, 2, 3, 1, 2, 3]); // Each sub gets all values
  });
  
  it('should demonstrate hot observable', () => {
    const values: number[] = [];
    const hot$ = new Subject<number>();
    
    hot$.subscribe(n => values.push(n));
    hot$.subscribe(n => values.push(n));
    
    hot$.next(1);
    hot$.next(2);
    
    expect(values).toEqual([1, 1, 2, 2]); // Both subs get each emission
  });
});
```

**Follow-up Questions:**

1. **When would you use ReplaySubject over BehaviorSubject?**
   - BehaviorSubject: Need current value, like state management
   - ReplaySubject: Need history, like event log
   - BehaviorSubject requires initial value, ReplaySubject doesn't

2. **How does shareReplay affect subscription lifecycle?**
   - Keeps subscription alive even when all subscribers unsubscribe
   - Can cause memory leaks if not careful
   - Use `shareReplay({ bufferSize: 1, refCount: true })` for proper cleanup

3. **Can you convert a hot observable to cold?**
   - Not really, by nature hot observables are already emitting
   - Can wrap in defer() to delay subscription
   - Better to design properly from the start

---

### Q3: How do you handle errors in RxJS streams, and what's the difference between catchError and retry?

**What the interviewer wants to know:**
- Error handling strategies
- Understanding of stream continuation
- Recovery patterns

**Strong Answer:**

Error handling in RxJS is crucial for building resilient applications. The key is understanding that errors terminate the observable stream unless properly handled.

**Error Termination:**

```typescript
// Without error handling - stream terminates
import { throwError, interval } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

interval(1000).pipe(
  mergeMap(n => {
    if (n === 3) {
      return throwError(() => new Error('Error at 3'));
    }
    return of(n);
  })
).subscribe({
  next: n => console.log('Value:', n),
  error: err => console.error('Error:', err),
  complete: () => console.log('Complete')
});

// Output:
// Value: 0
// Value: 1
// Value: 2
// Error: Error at 3
// Stream terminates, no more values
```

**catchError - Handle and Continue:**

```typescript
import { catchError } from 'rxjs/operators';
import { of, EMPTY } from 'rxjs';

// Pattern 1: Return fallback value
this.http.get<User>('/api/user/1').pipe(
  catchError(error => {
    console.error('Failed to load user:', error);
    // Return fallback value, stream continues
    return of({ id: 1, name: 'Guest User' } as User);
  })
).subscribe(user => {
  console.log('User:', user); // Either real user or guest
});

// Pattern 2: Return empty observable (complete immediately)
this.http.get<User[]>('/api/users').pipe(
  catchError(error => {
    console.error('Failed to load users:', error);
    return EMPTY; // Stream completes without emitting
  })
).subscribe({
  next: users => console.log('Users:', users),
  complete: () => console.log('Done') // Called immediately on error
});

// Pattern 3: Re-throw error after logging
this.http.post('/api/data', data).pipe(
  catchError(error => {
    console.error('Error:', error);
    this.logError(error);
    return throwError(() => error); // Re-throw to propagate
  })
).subscribe({
  next: result => console.log('Success:', result),
  error: err => this.showErrorMessage(err)
});
```

**catchError Placement Matters:**

```typescript
// Early catchError - handles inner observable errors
this.userIds$.pipe(
  mergeMap(id =>
    this.http.get(`/api/user/${id}`).pipe(
      catchError(error => {
        console.log(`Failed to load user ${id}`);
        return of(null); // Skip failed user, continue with others
      })
    )
  )
).subscribe(user => {
  if (user) {
    console.log('Loaded user:', user);
  }
});
// If one user fails, others still load

// Late catchError - handles entire stream errors
this.userIds$.pipe(
  mergeMap(id => this.http.get(`/api/user/${id}`))
).pipe(
  catchError(error => {
    console.log('Stream failed completely');
    return of([]); // Return empty array
  })
).subscribe(users => {
  console.log('Users:', users);
});
// Any error terminates entire stream
```

**retry - Automatic Retry:**

```typescript
import { retry, retryWhen, delay, take } from 'rxjs/operators';

// Simple retry - retry N times immediately
this.http.get('/api/data').pipe(
  retry(3) // Retry up to 3 times
).subscribe({
  next: data => console.log('Success:', data),
  error: err => console.error('Failed after 3 retries:', err)
});

// Retry with delay
this.http.get('/api/data').pipe(
  retryWhen(errors =>
    errors.pipe(
      delay(1000), // Wait 1 second between retries
      take(3) // Max 3 retries
    )
  )
).subscribe({
  next: data => console.log('Success:', data),
  error: err => console.error('Failed after retries:', err)
});

// Exponential backoff
this.http.get('/api/data').pipe(
  retryWhen(errors =>
    errors.pipe(
      mergeMap((error, index) => {
        if (index >= 3) {
          return throwError(() => error);
        }
        const backoff = Math.pow(2, index) * 1000;
        console.log(`Retry ${index + 1} after ${backoff}ms`);
        return timer(backoff);
      })
    )
  )
).subscribe({
  next: data => console.log('Success:', data),
  error: err => console.error('All retries failed:', err)
});
```

**Combining catchError and retry:**

```typescript
// retry first, then catchError
this.http.get<Data>('/api/data').pipe(
  retry(3), // Try 3 times
  catchError(error => {
    // If all retries fail, provide fallback
    console.error('All attempts failed:', error);
    return of({ default: true } as Data);
  })
).subscribe(data => {
  console.log('Data:', data); // Either real data or fallback
});
```

**Real-World Error Handling Patterns:**

**1. API Service with Comprehensive Error Handling:**

```typescript
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(
    private http: HttpClient,
    private errorHandler: ErrorHandlerService
  ) {}

  get<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retry({
        count: 2,
        delay: (error, retryCount) => {
          // Don't retry client errors (4xx)
          if (error.status >= 400 && error.status < 500) {
            return throwError(() => error);
          }
          // Exponential backoff for server errors
          return timer(Math.pow(2, retryCount) * 1000);
        }
      }),
      catchError(error => {
        this.errorHandler.log(error);
        
        // Specific error handling
        if (error.status === 404) {
          return throwError(() => new Error('Resource not found'));
        }
        if (error.status === 401) {
          this.router.navigate(['/login']);
          return EMPTY;
        }
        
        return throwError(() => new Error('An error occurred'));
      })
    );
  }
}
```

**2. Form Submission with Error Recovery:**

```typescript
@Component({
  selector: 'app-user-form'
})
export class UserFormComponent {
  submitForm(): void {
    const formData = this.form.value;
    
    this.userService.createUser(formData).pipe(
      tap(() => {
        this.showSuccess('User created successfully');
      }),
      catchError(error => {
        // Show error message to user
        if (error.status === 409) {
          this.showError('User already exists');
        } else if (error.status === 400) {
          this.showError('Invalid form data');
        } else {
          this.showError('Failed to create user');
        }
        
        // Don't propagate error, stream completes gracefully
        return EMPTY;
      }),
      finalize(() => {
        // Always runs, regardless of success or error
        this.form.enable();
        this.isSubmitting = false;
      })
    ).subscribe();
  }
}
```

**3. Multiple Parallel Requests with Partial Failure:**

```typescript
loadAllData(): void {
  const users$ = this.userService.getUsers().pipe(
    catchError(error => {
      console.error('Failed to load users:', error);
      return of([]); // Return empty array on failure
    })
  );
  
  const products$ = this.productService.getProducts().pipe(
    catchError(error => {
      console.error('Failed to load products:', error);
      return of([]); // Return empty array on failure
    })
  );
  
  const orders$ = this.orderService.getOrders().pipe(
    catchError(error => {
      console.error('Failed to load orders:', error);
      return of([]); // Return empty array on failure
    })
  );
  
  forkJoin({
    users: users$,
    products: products$,
    orders: orders$
  }).subscribe(({ users, products, orders }) => {
    // All three complete, even if some failed
    this.users = users;
    this.products = products;
    this.orders = orders;
    
    // Show partial success message if needed
    if (users.length === 0) {
      this.showWarning('Failed to load users');
    }
  });
}
```

**4. Polling with Error Handling:**

```typescript
startPolling(): void {
  interval(5000).pipe(
    switchMap(() =>
      this.dataService.getData().pipe(
        retry(1), // Retry once
        catchError(error => {
          console.error('Poll failed:', error);
          return of(null); // Continue polling despite error
        })
      )
    ),
    filter(data => data !== null), // Skip failed polls
    takeUntil(this.destroy$)
  ).subscribe(data => {
    this.updateData(data);
  });
}
```

**5. Chained Requests with Error Propagation:**

```typescript
createUserWithProfile(): void {
  this.userService.createUser(this.userData).pipe(
    // If user creation fails, stop here
    catchError(error => {
      this.showError('Failed to create user');
      return throwError(() => error);
    }),
    
    // Only runs if user creation succeeds
    switchMap(user =>
      this.profileService.createProfile(user.id, this.profileData).pipe(
        // If profile creation fails, delete user (rollback)
        catchError(profileError => {
          return this.userService.deleteUser(user.id).pipe(
            switchMap(() => throwError(() => profileError))
          );
        })
      )
    ),
    
    // Final success
    tap(() => {
      this.showSuccess('User and profile created');
      this.router.navigate(['/users']);
    }),
    
    // Handle any remaining errors
    catchError(error => {
      this.showError('Operation failed');
      return EMPTY;
    })
  ).subscribe();
}
```

**Error Recovery Strategies:**

```typescript
// Strategy 1: Graceful degradation
this.getPremiumFeatures().pipe(
  catchError(() => this.getBasicFeatures())
).subscribe(features => {
  this.features = features;
});

// Strategy 2: Cache fallback
this.http.get('/api/data').pipe(
  catchError(() => this.cacheService.getCachedData())
).subscribe(data => {
  this.data = data;
});

// Strategy 3: User retry
this.loadData().pipe(
  catchError(error => {
    return this.confirmRetry().pipe(
      switchMap(shouldRetry =>
        shouldRetry ? this.loadData() : throwError(() => error)
      )
    );
  })
).subscribe();
```

**Follow-up Questions:**

1. **What's the difference between catchError and tap for errors?**
   - tap: Side effects only, doesn't handle errors (stream still terminates)
   - catchError: Handles errors, can return fallback, continues stream
   - tap is for logging, catchError is for recovery

2. **How do you handle errors in nested observables?**
   - Place catchError in inner observable to handle individually
   - Place catchError in outer observable to handle all
   - Choice depends on whether you want partial or complete failure

3. **What happens if catchError throws an error?**
   - New error propagates downstream
   - Stream terminates with new error
   - Useful for error transformation

---

## Advanced Questions

### Q4: Explain subscription management and memory leak prevention strategies.

**What the interviewer wants to know:**
- Understanding of subscription lifecycle
- Memory leak awareness
- Best practices for cleanup

**Strong Answer:**

Proper subscription management is critical to prevent memory leaks in Angular applications. Unmanaged subscriptions can cause components to remain in memory after destruction.

**The Problem:**

```typescript
// ❌ MEMORY LEAK
@Component({
  selector: 'app-leaky'
})
export class LeakyComponent implements OnInit {
  ngOnInit(): void {
    // Subscription never cleaned up!
    interval(1000).subscribe(n => {
      console.log('Tick:', n);
    });
    
    this.dataService.getData().subscribe(data => {
      console.log('Data:', data);
    });
  }
}

// When component is destroyed, subscriptions remain active
// Console continues logging even after navigation away
// Memory is never freed
```

**Solution 1: Manual Unsubscription:**

```typescript
@Component({
  selector: 'app-manual'
})
export class ManualComponent implements OnInit, OnDestroy {
  private subscription = new Subscription();
  
  ngOnInit(): void {
    // Add subscriptions to Subscription container
    this.subscription.add(
      interval(1000).subscribe(n => console.log('Tick:', n))
    );
    
    this.subscription.add(
      this.dataService.getData().subscribe(data => {
        console.log('Data:', data);
      })
    );
  }
  
  ngOnDestroy(): void {
    // Unsubscribe from all at once
    this.subscription.unsubscribe();
  }
}
```

**Solution 2: takeUntil Pattern (Recommended):**

```typescript
@Component({
  selector: 'app-takeuntil'
})
export class TakeUntilComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // All subscriptions automatically complete when destroy$ emits
    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(n => console.log('Tick:', n));
    
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => console.log('Data:', data));
    
    this.userService.getUser().pipe(
      takeUntil(this.destroy$)
    ).subscribe(user => this.user = user);
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Solution 3: Async Pipe (Best for Templates):**

```typescript
@Component({
  selector: 'app-async',
  template: `
    <div *ngIf="user$ | async as user">
      {{ user.name }}
    </div>
    
    <div *ngFor="let item of items$ | async">
      {{ item.name }}
    </div>
    
    <p>Count: {{ count$ | async }}</p>
  `
})
export class AsyncComponent {
  // No manual subscription needed
  user$ = this.userService.getUser();
  items$ = this.itemService.getItems();
  count$ = interval(1000);
  
  // Async pipe automatically unsubscribes when component destroys
  constructor(
    private userService: UserService,
    private itemService: ItemService
  ) {}
}
```

**Solution 4: take(1) for Single Emissions:**

```typescript
@Component({
  selector: 'app-single'
})
export class SingleEmissionComponent {
  loadUser(): void {
    // Automatically completes after first emission
    this.userService.getUser().pipe(
      take(1)
    ).subscribe(user => {
      this.user = user;
    });
    // No manual cleanup needed
  }
  
  submitForm(): void {
    this.formService.submit(this.formData).pipe(
      take(1)
    ).subscribe(
      result => this.handleSuccess(result),
      error => this.handleError(error)
    );
  }
}
```

**Comparison of Strategies:**

```typescript
@Component({
  selector: 'app-comparison'
})
export class ComparisonComponent implements OnInit, OnDestroy {
  // Manual subscription management
  private subscriptions = new Subscription();
  
  // takeUntil pattern
  private destroy$ = new Subject<void>();
  
  // Async pipe - no manual management
  data$ = this.dataService.getData();
  
  ngOnInit(): void {
    // Pattern 1: Manual
    const sub1 = this.service.getData().subscribe();
    this.subscriptions.add(sub1);
    
    // Pattern 2: takeUntil
    this.service.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe();
    
    // Pattern 3: take(1) - single emission
    this.service.getData().pipe(
      take(1)
    ).subscribe();
    
    // Pattern 4: Async pipe (in template)
    // this.data$ - automatically cleaned up
  }
  
  ngOnDestroy(): void {
    this.subscriptions.unsubscribe();
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Real-World Examples:**

**1. Form Value Changes:**

```typescript
@Component({
  selector: 'app-search'
})
export class SearchComponent implements OnInit, OnDestroy {
  searchControl = new FormControl('');
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // Search as user types
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    ).subscribe(query => {
      this.performSearch(query);
    });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**2. Route Parameters:**

```typescript
@Component({
  selector: 'app-user-detail'
})
export class UserDetailComponent implements OnInit, OnDestroy {
  user$: Observable<User>;
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // Option 1: takeUntil
    this.route.params.pipe(
      switchMap(params => this.userService.getUser(params['id'])),
      takeUntil(this.destroy$)
    ).subscribe(user => {
      this.user = user;
    });
    
    // Option 2: Async pipe (preferred)
    this.user$ = this.route.params.pipe(
      switchMap(params => this.userService.getUser(params['id']))
    );
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**3. WebSocket Connection:**

```typescript
@Component({
  selector: 'app-realtime'
})
export class RealtimeComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    this.wsService.connect().pipe(
      takeUntil(this.destroy$)
    ).subscribe(message => {
      this.handleMessage(message);
    });
  }
  
  ngOnDestroy(): void {
    // Properly closes WebSocket connection
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**4. Interval/Timer Cleanup:**

```typescript
@Component({
  selector: 'app-timer'
})
export class TimerComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // Polling every 5 seconds
    interval(5000).pipe(
      switchMap(() => this.dataService.getData()),
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.refreshData(data);
    });
  }
  
  ngOnDestroy(): void {
    // Stops polling
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Advanced Patterns:**

**1. Conditional Subscription:**

```typescript
@Component({
  selector: 'app-conditional'
})
export class ConditionalComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private stopPolling$ = new Subject<void>();
  
  ngOnInit(): void {
    // Start polling
    interval(5000).pipe(
      takeUntil(merge(this.destroy$, this.stopPolling$))
    ).subscribe(() => {
      this.poll();
    });
  }
  
  stopPolling(): void {
    // Stop polling without destroying component
    this.stopPolling$.next();
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**2. Subscription Reuse:**

```typescript
@Component({
  selector: 'app-reuse'
})
export class ReuseComponent {
  private currentSub: Subscription | null = null;
  
  loadData(id: number): void {
    // Cancel previous subscription
    if (this.currentSub) {
      this.currentSub.unsubscribe();
    }
    
    // Create new subscription
    this.currentSub = this.dataService.getData(id).subscribe(data => {
      this.data = data;
    });
  }
  
  ngOnDestroy(): void {
    if (this.currentSub) {
      this.currentSub.unsubscribe();
    }
  }
}

// Better with switchMap
@Component({
  selector: 'app-reuse-better'
})
export class ReuseBetterComponent implements OnDestroy {
  private loadData$ = new Subject<number>();
  private destroy$ = new Subject<void>();
  
  constructor() {
    this.loadData$.pipe(
      switchMap(id => this.dataService.getData(id)),
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.data = data;
    });
  }
  
  loadData(id: number): void {
    this.loadData$.next(id); // Automatically cancels previous
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**3. Service Subscriptions:**

```typescript
@Injectable({ providedIn: 'root' })
export class DataPollingService implements OnDestroy {
  private polling$ = interval(5000).pipe(
    switchMap(() => this.http.get('/api/data')),
    shareReplay(1) // Share among all subscribers
  );
  
  private subscriptions = new Subscription();
  
  constructor(private http: HttpClient) {
    // Service subscriptions need cleanup too
    this.subscriptions.add(
      this.polling$.subscribe()
    );
  }
  
  getData(): Observable<any> {
    return this.polling$;
  }
  
  ngOnDestroy(): void {
    this.subscriptions.unsubscribe();
  }
}
```

**Testing Subscription Cleanup:**

```typescript
describe('SubscriptionComponent', () => {
  let component: SubscriptionComponent;
  let fixture: ComponentFixture<SubscriptionComponent>;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [SubscriptionComponent]
    });
    fixture = TestBed.createComponent(SubscriptionComponent);
    component = fixture.componentInstance;
  });
  
  it('should unsubscribe on destroy', () => {
    const spy = jasmine.createSpy('subscription');
    const observable = interval(1000);
    
    component.ngOnInit();
    component['subscription'] = observable.subscribe(spy);
    
    fixture.destroy();
    
    // Wait to ensure no more emissions
    setTimeout(() => {
      expect(spy).toHaveBeenCalledTimes(0);
    }, 2000);
  });
});
```

**Follow-up Questions:**

1. **When would you NOT need to unsubscribe?**
   - HTTP requests (complete automatically)
   - take(1) operator
   - first() operator
   - Async pipe (handles cleanup)
   - Finite observables (of, from)

2. **What's the performance impact of many subscriptions?**
   - Each subscription consumes memory
   - Event listeners remain active
   - Can cause slow-downs with hundreds of subscriptions
   - Leads to memory leaks if not cleaned up

3. **How do you debug subscription leaks?**
   - Angular DevTools (check for components not destroyed)
   - Chrome Memory Profiler
   - Add logging in ngOnDestroy
   - Track subscription count
   - Use takeUntil consistently

---

## What Interviewers Look For

### Strong Signals

1. **Operator Mastery**: Knows which operator to use when
2. **Error Handling**: Comprehensive error management strategies
3. **Memory Management**: Understands subscription lifecycle
4. **Real Experience**: Shares specific project examples
5. **Performance Awareness**: Discusses hot vs cold, sharing strategies
6. **Best Practices**: Uses async pipe, takeUntil pattern
7. **Testing Knowledge**: Can test observable streams

## Red Flags to Avoid

1. **"Just use async pipe for everything"**: Oversimplification
2. **Confusing map with mergeMap**: Fundamental misunderstanding
3. **No subscription cleanup**: Memory leak issues
4. **"Never used retry or catchError"**: Limited production experience
5. **Can't explain hot vs cold**: Conceptual gap
6. **No testing experience**: Incomplete skill set
7. **Overusing subscriptions**: Not leveraging reactive patterns

## Key Takeaways

### Essential Concepts

1. **Flattening Operators**
   - map: Transform values
   - mergeMap: Concurrent, no cancellation
   - switchMap: Cancel previous, latest only
   - concatMap: Sequential, ordered
   - exhaustMap: Ignore while busy

2. **Hot vs Cold**
   - Cold: New execution per subscription
   - Hot: Shared execution
   - Use share/shareReplay to convert

3. **Error Handling**
   - catchError: Handle and continue
   - retry: Automatic retry logic
   - Combine for robust error recovery

4. **Subscription Management**
   - Async pipe (preferred)
   - takeUntil pattern
   - Manual unsubscribe
   - take(1) for single emissions

### Best Practices

1. Use switchMap for search/navigation
2. Use async pipe when possible
3. Implement takeUntil pattern consistently
4. Handle errors at appropriate level
5. Use shareReplay for caching
6. Test observable streams
7. Avoid nested subscriptions

### Interview Success Tips

1. **Start with basics**: Explain observables clearly
2. **Show operator knowledge**: When to use each
3. **Discuss real scenarios**: Share project examples
4. **Explain trade-offs**: No perfect solution
5. **Memory awareness**: Always discuss cleanup
6. **Error handling**: Show robust patterns
7. **Ask clarifying questions**: Understand requirements

### Quick Reference

```typescript
// Operators
map(x => x * 2)
mergeMap(x => api.get(x))
switchMap(x => api.get(x))
concatMap(x => api.post(x))
exhaustMap(() => api.submit())

// Error handling
catchError(err => of(fallback))
retry(3)
retryWhen(errors => errors.pipe(delay(1000)))

// Subscription cleanup
takeUntil(this.destroy$)
take(1)
async pipe

// Sharing
share()
shareReplay(1)
shareReplay({ bufferSize: 1, refCount: true })
```

Remember: RxJS mastery comes with practice. Start with common patterns, build understanding through real projects, and gradually tackle advanced scenarios.
