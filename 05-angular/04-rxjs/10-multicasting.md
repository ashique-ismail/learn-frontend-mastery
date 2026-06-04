# RxJS Multicasting

## Table of Contents
- [Introduction](#introduction)
- [Unicast vs Multicast](#unicast-vs-multicast)
- [share Operator](#share-operator)
- [shareReplay](#sharereplay)
- [publish and connect](#publish-and-connect)
- [Subjects for Multicasting](#subjects-for-multicasting)
- [Real-World Patterns](#real-world-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Multicasting is a technique in RxJS where a single observable execution is shared among multiple subscribers. By default, observables are **unicast** - each subscriber triggers a separate execution. Multicasting makes observables **multicast** - all subscribers share the same execution.

Understanding multicasting is crucial for performance optimization, preventing duplicate HTTP requests, and managing shared state in Angular applications. It's especially important when dealing with expensive operations like HTTP calls, WebSocket connections, or complex calculations.

## Unicast vs Multicast

### Unicast (Default Behavior)

In unicast observables, each subscriber gets its own independent execution.

```typescript
import { Observable } from 'rxjs';

// Unicast: Each subscriber triggers separate execution
const unicast$ = new Observable(observer => {
  console.log('Observable execution started');
  observer.next(Math.random());
  observer.complete();
});

// First subscriber
unicast$.subscribe(value => console.log('Subscriber 1:', value));
// Output: "Observable execution started"
//         "Subscriber 1: 0.123..."

// Second subscriber
unicast$.subscribe(value => console.log('Subscriber 2:', value));
// Output: "Observable execution started"
//         "Subscriber 2: 0.456..."
// Different random number! Separate execution.
```

### HTTP Request Example (Unicast Problem)

```typescript
@Component({
  selector: 'app-user-profile'
})
export class UserProfileComponent implements OnInit {
  private http = inject(HttpClient);
  
  // PROBLEM: Each subscription makes a new HTTP request
  user$ = this.http.get<User>('/api/user');

  ngOnInit() {
    // First HTTP request
    this.user$.subscribe(user => console.log('Name:', user.name));
    
    // Second HTTP request (duplicate!)
    this.user$.subscribe(user => console.log('Email:', user.email));
    
    // Third HTTP request (another duplicate!)
    this.user$.subscribe(user => console.log('Age:', user.age));
  }
}
```

### Multicast Solution

```typescript
import { shareReplay } from 'rxjs/operators';

@Component({
  selector: 'app-user-profile'
})
export class UserProfileComponent implements OnInit {
  private http = inject(HttpClient);
  
  // SOLUTION: Share single HTTP request across all subscribers
  user$ = this.http.get<User>('/api/user').pipe(
    shareReplay(1) // Cache last value, share execution
  );

  ngOnInit() {
    // Only ONE HTTP request for all three subscriptions
    this.user$.subscribe(user => console.log('Name:', user.name));
    this.user$.subscribe(user => console.log('Email:', user.email));
    this.user$.subscribe(user => console.log('Age:', user.age));
  }
}
```

## share Operator

The `share` operator converts a cold observable into a hot observable that's shared among multiple subscribers.

### Basic Usage

```typescript
import { share } from 'rxjs/operators';
import { interval } from 'rxjs';

@Component({
  selector: 'app-shared-timer'
})
export class SharedTimerComponent implements OnInit {
  // Without share: Each subscriber gets its own timer
  private coldTimer$ = interval(1000);

  // With share: All subscribers share the same timer
  sharedTimer$ = interval(1000).pipe(
    share()
  );

  ngOnInit() {
    // Both subscribers receive the same values at the same time
    this.sharedTimer$.subscribe(val => console.log('Sub 1:', val));
    
    setTimeout(() => {
      this.sharedTimer$.subscribe(val => console.log('Sub 2:', val));
    }, 2500);
    
    // Sub 1: 0, 1, 2, 3...
    // Sub 2: starts at 2 (joins existing stream)
  }
}
```

### Real-World Example: Shared Data Loading

```typescript
@Injectable()
export class ProductService {
  constructor(private http: HttpClient) {}

  private productsSource$ = this.http.get<Product[]>('/api/products').pipe(
    share() // Share the HTTP request
  );

  getProducts(): Observable<Product[]> {
    return this.productsSource$;
  }

  getProductCount(): Observable<number> {
    return this.productsSource$.pipe(
      map(products => products.length)
    );
  }

  getExpensiveProducts(): Observable<Product[]> {
    return this.productsSource$.pipe(
      map(products => products.filter(p => p.price > 100))
    );
  }
}

// In component - only ONE HTTP request for all three subscriptions
@Component({
  selector: 'app-product-dashboard'
})
export class ProductDashboardComponent {
  private productService = inject(ProductService);

  products$ = this.productService.getProducts();
  productCount$ = this.productService.getProductCount();
  expensiveProducts$ = this.productService.getExpensiveProducts();
}
```

### share Configuration (RxJS 7+)

```typescript
import { share, ShareConfig } from 'rxjs';

@Injectable()
export class ConfigurableShareService {
  constructor(private http: HttpClient) {}

  getData() {
    const config: ShareConfig<any> = {
      connector: () => new Subject(), // Factory for sharing
      resetOnError: true,             // Reset on error
      resetOnComplete: false,          // Don't reset on complete
      resetOnRefCountZero: true       // Reset when no subscribers
    };

    return this.http.get('/api/data').pipe(
      share(config)
    );
  }
}
```

## shareReplay

`shareReplay` shares the observable execution AND replays the specified number of emissions to new subscribers.

### Basic Usage

```typescript
import { shareReplay } from 'rxjs/operators';

@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}

  private currentUser$ = this.http.get<User>('/api/user/current').pipe(
    shareReplay(1) // Cache last emission
  );

  getCurrentUser(): Observable<User> {
    return this.currentUser$;
  }
}

// First subscriber triggers HTTP request
service.getCurrentUser().subscribe(user => console.log('User 1:', user));

// Second subscriber gets cached value (no new HTTP request)
setTimeout(() => {
  service.getCurrentUser().subscribe(user => console.log('User 2:', user));
}, 5000);
```

### Cache Invalidation

```typescript
@Injectable()
export class CacheableDataService {
  constructor(private http: HttpClient) {}

  private dataSubject = new BehaviorSubject<number>(0);
  
  private data$ = this.dataSubject.pipe(
    switchMap(version => 
      this.http.get<Data>('/api/data').pipe(
        shareReplay(1)
      )
    )
  );

  getData(): Observable<Data> {
    return this.data$;
  }

  refreshData(): void {
    // Increment version to trigger new request
    this.dataSubject.next(this.dataSubject.value + 1);
  }
}
```

### shareReplay Configuration

```typescript
import { shareReplay, ShareReplayConfig } from 'rxjs';

@Injectable()
export class ConfigService {
  constructor(private http: HttpClient) {}

  private config$ = this.http.get<Config>('/api/config').pipe(
    shareReplay({
      bufferSize: 1,           // Number of values to replay
      refCount: true,          // Unsubscribe from source when no subscribers
      windowTime: 30000        // Cache duration (30 seconds)
    })
  );

  getConfig(): Observable<Config> {
    return this.config$;
  }
}
```

### Comparison: share vs shareReplay

```typescript
@Injectable()
export class ComparisonService {
  constructor(private http: HttpClient) {}

  // share: New subscribers don't get previous values
  dataWithShare$ = this.http.get('/api/data').pipe(
    share()
  );

  // shareReplay: New subscribers get cached last value
  dataWithShareReplay$ = this.http.get('/api/data').pipe(
    shareReplay(1)
  );
}

// share example
service.dataWithShare$.subscribe(d => console.log('Sub 1:', d)); // Gets data
setTimeout(() => {
  service.dataWithShare$.subscribe(d => console.log('Sub 2:', d)); // Waits for next emission
}, 1000);

// shareReplay example
service.dataWithShareReplay$.subscribe(d => console.log('Sub 1:', d)); // Gets data
setTimeout(() => {
  service.dataWithShareReplay$.subscribe(d => console.log('Sub 2:', d)); // Gets cached data immediately
}, 1000);
```

## publish and connect

Lower-level operators that give you manual control over when the shared observable starts.

### Basic publish and connect

```typescript
import { publish, ConnectableObservable } from 'rxjs';

@Component({
  selector: 'app-manual-connect'
})
export class ManualConnectComponent implements OnInit {
  ngOnInit() {
    const source$ = interval(1000).pipe(
      take(5),
      tap(x => console.log('Source emits:', x))
    ) as ConnectableObservable<number>;

    const multicasted$ = source$.pipe(publish());

    // Subscribe but execution doesn't start yet
    multicasted$.subscribe(x => console.log('Subscriber 1:', x));
    multicasted$.subscribe(x => console.log('Subscriber 2:', x));

    // Manually start execution
    const subscription = (multicasted$ as ConnectableObservable<number>).connect();

    // Can unsubscribe from source later
    setTimeout(() => subscription.unsubscribe(), 3000);
  }
}
```

### refCount

```typescript
import { publish, refCount } from 'rxjs/operators';

@Injectable()
export class RefCountService {
  constructor(private http: HttpClient) {}

  // Automatically connects when first subscriber arrives
  // Automatically disconnects when last subscriber leaves
  data$ = this.http.get('/api/data').pipe(
    publish(),
    refCount()
  );

  // Note: share() is equivalent to publish() + refCount()
  dataSame$ = this.http.get('/api/data').pipe(
    share() // Same as above
  );
}
```

## Subjects for Multicasting

Subjects can be used to manually create multicast observables.

### Manual Multicasting with Subject

```typescript
@Injectable()
export class EventBusService {
  private eventSubject = new Subject<AppEvent>();

  // Expose as observable (multicast by nature)
  events$ = this.eventSubject.asObservable();

  emit(event: AppEvent): void {
    this.eventSubject.next(event);
  }
}

// Multiple components can subscribe
@Component({
  selector: 'app-listener-1'
})
export class Listener1Component implements OnInit {
  private eventBus = inject(EventBusService);

  ngOnInit() {
    this.eventBus.events$.subscribe(event => {
      console.log('Listener 1 received:', event);
    });
  }
}
```

### BehaviorSubject for Cached State

```typescript
@Injectable()
export class StateService {
  private stateSubject = new BehaviorSubject<AppState>({
    user: null,
    isLoading: false
  });

  // Multicast with replay of current value
  state$ = this.stateSubject.asObservable();

  updateState(newState: Partial<AppState>): void {
    this.stateSubject.next({
      ...this.stateSubject.value,
      ...newState
    });
  }

  getState(): AppState {
    return this.stateSubject.value;
  }
}
```

### ReplaySubject for Multiple Values

```typescript
@Injectable()
export class ActivityLogService {
  // Replay last 10 activities to new subscribers
  private logSubject = new ReplaySubject<Activity>(10);

  activities$ = this.logSubject.asObservable();

  logActivity(activity: Activity): void {
    this.logSubject.next(activity);
  }
}
```

## Real-World Patterns

### Pattern 1: Shared HTTP Call with Loading State

```typescript
interface DataState<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

@Injectable()
export class DataStateService {
  constructor(private http: HttpClient) {}

  private dataState$ = new BehaviorSubject<DataState<User>>({
    data: null,
    loading: false,
    error: null
  });

  state$ = this.dataState$.asObservable();

  loadData(): void {
    this.dataState$.next({ data: null, loading: true, error: null });

    this.http.get<User>('/api/user').pipe(
      shareReplay(1) // Share result among multiple subscribers
    ).subscribe({
      next: data => {
        this.dataState$.next({ data, loading: false, error: null });
      },
      error: err => {
        this.dataState$.next({ 
          data: null, 
          loading: false, 
          error: err.message 
        });
      }
    });
  }
}
```

### Pattern 2: WebSocket Connection Sharing

```typescript
@Injectable()
export class WebSocketService {
  private socket$: Observable<any> | null = null;

  connect(url: string): Observable<any> {
    if (!this.socket$) {
      this.socket$ = new Observable(observer => {
        const ws = new WebSocket(url);

        ws.onmessage = event => observer.next(event.data);
        ws.onerror = err => observer.error(err);
        ws.onclose = () => observer.complete();

        return () => ws.close();
      }).pipe(
        share() // Share WebSocket connection
      );
    }

    return this.socket$;
  }

  disconnect(): void {
    this.socket$ = null;
  }
}

// Multiple components can share the same WebSocket connection
@Component({
  selector: 'app-live-feed'
})
export class LiveFeedComponent {
  private ws = inject(WebSocketService);

  messages$ = this.ws.connect('wss://api.example.com/feed');
}
```

### Pattern 3: Polling with Shared Results

```typescript
@Injectable()
export class PollingService {
  constructor(private http: HttpClient) {}

  poll(url: string, intervalMs: number = 5000): Observable<any> {
    return interval(intervalMs).pipe(
      startWith(0),
      switchMap(() => this.http.get(url)),
      shareReplay(1) // Share polling results
    );
  }
}

// Multiple components get same polled data
@Component({
  selector: 'app-dashboard'
})
export class DashboardComponent {
  private polling = inject(PollingService);

  serverStatus$ = this.polling.poll('/api/status', 3000);
}
```

### Pattern 4: Debounced Search with Caching

```typescript
@Injectable()
export class SearchService {
  constructor(private http: HttpClient) {}

  private searchCache = new Map<string, Observable<any>>();

  search(query: string): Observable<any> {
    if (!this.searchCache.has(query)) {
      const search$ = this.http.get(`/api/search?q=${query}`).pipe(
        shareReplay({
          bufferSize: 1,
          refCount: true,
          windowTime: 300000 // 5 minute cache
        })
      );
      
      this.searchCache.set(query, search$);
    }

    return this.searchCache.get(query)!;
  }

  clearCache(): void {
    this.searchCache.clear();
  }
}
```

### Pattern 5: Multi-Component Data Sharing

```typescript
@Injectable()
export class SharedDataService {
  constructor(private http: HttpClient) {}

  private refreshTrigger$ = new BehaviorSubject<void>(undefined);

  // Shared data stream that refreshes on demand
  data$ = this.refreshTrigger$.pipe(
    switchMap(() => 
      this.http.get<Data>('/api/data').pipe(
        shareReplay(1)
      )
    )
  );

  refresh(): void {
    this.refreshTrigger$.next();
  }
}

// Component 1
@Component({
  selector: 'app-viewer-1'
})
export class Viewer1Component {
  private shared = inject(SharedDataService);
  data$ = this.shared.data$; // Shares HTTP request with Viewer2
}

// Component 2
@Component({
  selector: 'app-viewer-2'
})
export class Viewer2Component {
  private shared = inject(SharedDataService);
  data$ = this.shared.data$; // Gets same data as Viewer1
}
```

## Common Mistakes

### 1. Memory Leaks with shareReplay

```typescript
// WRONG: shareReplay without refCount keeps source subscribed forever
const leak$ = this.http.get('/api/data').pipe(
  shareReplay(1) // Stays subscribed even with no subscribers!
);

// RIGHT: Use refCount to clean up
const safe$ = this.http.get('/api/data').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

### 2. Not Understanding Cold vs Hot

```typescript
// WRONG: Thinking share makes cold observable hot immediately
const timer$ = interval(1000).pipe(share());
// Nothing happens yet - still need a subscriber

// RIGHT: Understand that share needs at least one subscriber
timer$.subscribe(); // Now it's active
```

### 3. Using shareReplay on Long-Lived Streams

```typescript
// WRONG: shareReplay on infinite stream
const clicks$ = fromEvent(button, 'click').pipe(
  shareReplay(1) // Will cache forever!
);

// RIGHT: Use share for event streams
const clicks$ = fromEvent(button, 'click').pipe(
  share()
);
```

### 4. Forgetting Cache Invalidation

```typescript
// WRONG: Cached data never updates
@Injectable()
export class CachedService {
  data$ = this.http.get('/api/data').pipe(
    shareReplay(1) // Cached forever
  );
}

// RIGHT: Provide refresh mechanism
@Injectable()
export class RefreshableService {
  private refresh$ = new Subject<void>();
  
  data$ = this.refresh$.pipe(
    startWith(undefined),
    switchMap(() => this.http.get('/api/data')),
    shareReplay(1)
  );

  refresh() {
    this.refresh$.next();
  }
}
```

## Best Practices

### 1. Use shareReplay for HTTP Calls

```typescript
// Good pattern for caching HTTP responses
getData(): Observable<Data> {
  return this.http.get<Data>('/api/data').pipe(
    shareReplay({ bufferSize: 1, refCount: true })
  );
}
```

### 2. Use share for Event Streams

```typescript
// Good pattern for sharing event streams
clicks$ = fromEvent(button, 'click').pipe(
  share() // Don't cache clicks
);
```

### 3. Provide Manual Refresh

```typescript
@Injectable()
export class RefreshableDataService {
  private refresh$ = new Subject<void>();

  data$ = this.refresh$.pipe(
    startWith(undefined),
    switchMap(() => this.http.get('/api/data')),
    shareReplay(1)
  );

  refresh(): void {
    this.refresh$.next();
  }
}
```

### 4. Document Multicast Behavior

```typescript
@Injectable()
export class DocumentedService {
  /**
   * Gets user data.
   * NOTE: This observable is multicast - multiple subscriptions
   * share the same HTTP request. Data is cached for 5 minutes.
   */
  getUser(): Observable<User> {
    return this.http.get<User>('/api/user').pipe(
      shareReplay({
        bufferSize: 1,
        refCount: true,
        windowTime: 300000
      })
    );
  }
}
```

## Interview Questions

### Q1: What's the difference between share and shareReplay?

**Answer:**
- **share**: Multicasts to all current subscribers. New subscribers don't get previous values.
- **shareReplay**: Multicasts AND caches specified number of values. New subscribers get cached values immediately.

Use `share` for events, `shareReplay` for data that should be cached.

### Q2: What is the refCount option in shareReplay?

**Answer:** `refCount: true` makes the observable unsubscribe from its source when the reference count drops to zero (no subscribers). Without it, the source stays subscribed forever, causing memory leaks. Always use `refCount: true` for finite observables like HTTP requests.

### Q3: How does shareReplay cause memory leaks?

**Answer:** By default, `shareReplay()` keeps the source observable subscribed even when there are no subscribers. For HTTP requests that complete, this isn't usually a problem, but for long-lived streams it leaks memory. Use `shareReplay({ refCount: true })` to prevent this.

## Key Takeaways

1. Observables are **unicast** by default - each subscriber gets separate execution
2. **Multicasting** shares one execution among multiple subscribers
3. **share** - Multicast without caching
4. **shareReplay** - Multicast with caching
5. Use `shareReplay({ refCount: true })` to prevent memory leaks
6. Share HTTP requests to avoid duplicate calls
7. Don't cache event streams with shareReplay
8. Provide refresh mechanisms for cached data
9. Subjects are naturally multicast
10. Document multicast behavior in services

## Resources

- [RxJS Documentation - Multicasting](https://rxjs.dev/guide/operators#multicasting-operators)
- [Learn RxJS - Multicasting](https://www.learnrxjs.io/learn-rxjs/operators/multicasting)
- [Ben Lesh - Hot vs Cold Observables](https://medium.com/@benlesh/hot-vs-cold-observables-f8094ed53339)
- [Angular University - shareReplay](https://blog.angular-university.io/how-to-build-angular2-apps-using-rxjs-observable-data-services-pitfalls-to-avoid/)
