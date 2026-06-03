# Hot vs Cold Observables

## Table of Contents
- [Introduction](#introduction)
- [Cold Observables](#cold-observables)
- [Hot Observables](#hot-observables)
- [Key Differences](#key-differences)
- [Converting Cold to Hot](#converting-cold-to-hot)
- [When to Use Each](#when-to-use-each)
- [Real-World Examples](#real-world-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Understanding the distinction between hot and cold observables is fundamental to mastering RxJS in Angular. This concept affects performance, behavior, and the architecture of your reactive applications. The difference lies in **when** and **how** the observable produces values in relation to its subscribers.

Think of it like streaming services:
- **Cold Observable**: Like a Netflix movie - starts from the beginning for each viewer
- **Hot Observable**: Like live TV - you join what's already broadcasting

## Cold Observables

Cold observables are **unicast** - they create a new producer for each subscriber. The data production starts when you subscribe.

### Characteristics

1. **Passive**: Doesn't produce values until subscribed
2. **Unicast**: Each subscriber gets its own independent execution
3. **Replay from start**: Each subscriber gets all values from the beginning
4. **Independent**: Subscribers don't affect each other

### Basic Example

```typescript
import { Observable } from 'rxjs';

// Cold observable - creates new producer per subscriber
const cold$ = new Observable(observer => {
  console.log('Producer started');
  const value = Math.random();
  observer.next(value);
  observer.complete();
});

console.log('Before subscriptions');

// First subscription - starts producer
cold$.subscribe(value => console.log('Sub 1:', value));
// Output: "Producer started"
//         "Sub 1: 0.123..."

// Second subscription - starts NEW producer
cold$.subscribe(value => console.log('Sub 2:', value));
// Output: "Producer started"
//         "Sub 2: 0.456..." (different value!)
```

### HTTP Requests (Cold)

```typescript
@Injectable()
export class ProductService {
  constructor(private http: HttpClient) {}

  // Cold observable - each subscription makes a new HTTP request
  getProduct(id: string): Observable<Product> {
    console.log('Creating HTTP observable');
    return this.http.get<Product>(`/api/products/${id}`);
  }
}

@Component({
  selector: 'app-product-viewer'
})
export class ProductViewerComponent {
  private productService = inject(ProductService);

  ngOnInit() {
    const product$ = this.productService.getProduct('123');

    // First subscription - HTTP request #1
    product$.subscribe(p => console.log('Name:', p.name));

    // Second subscription - HTTP request #2 (duplicate!)
    product$.subscribe(p => console.log('Price:', p.price));

    // Problem: Two identical HTTP requests for the same data!
  }
}
```

### Intervals and Timers (Cold)

```typescript
import { interval } from 'rxjs';

// Cold observable - each subscriber gets its own timer
const timer$ = interval(1000);

console.log('Starting subscriptions');

// Subscriber 1 starts at 0
timer$.subscribe(val => console.log('Timer 1:', val));

// Subscriber 2 starts 2.5 seconds later, but also starts at 0
setTimeout(() => {
  timer$.subscribe(val => console.log('Timer 2:', val));
}, 2500);

// Output:
// Timer 1: 0
// Timer 1: 1
// Timer 1: 2
// Timer 2: 0  <- Starts from 0, not 2!
// Timer 1: 3
// Timer 2: 1
```

### Custom Cold Observable

```typescript
@Injectable()
export class DataGeneratorService {
  // Cold - generates new data sequence for each subscriber
  generateData(): Observable<number> {
    return new Observable(observer => {
      console.log('Starting data generation');
      let count = 0;

      const interval = setInterval(() => {
        observer.next(count++);
        if (count > 5) {
          clearInterval(interval);
          observer.complete();
        }
      }, 1000);

      // Cleanup
      return () => {
        console.log('Cleaning up');
        clearInterval(interval);
      };
    });
  }
}

// Each subscriber gets independent data generation
const data$ = service.generateData();
data$.subscribe(val => console.log('Sub 1:', val)); // 0, 1, 2, 3, 4, 5
data$.subscribe(val => console.log('Sub 2:', val)); // 0, 1, 2, 3, 4, 5
```

## Hot Observables

Hot observables are **multicast** - they share a single producer among all subscribers. The producer is active regardless of subscriptions.

### Characteristics

1. **Active**: Produces values regardless of subscribers
2. **Multicast**: All subscribers share the same execution
3. **Join in progress**: Subscribers get values from subscription point onward
4. **Shared**: Subscribers affect each other (shared execution)

### Basic Example with Subject

```typescript
import { Subject } from 'rxjs';

// Hot observable - single shared producer
const hot$ = new Subject<number>();

console.log('Before subscriptions');

// First subscriber
hot$.subscribe(value => console.log('Sub 1:', value));

// Emit values (both subscribers will receive)
hot$.next(1);
hot$.next(2);

// Second subscriber joins
hot$.subscribe(value => console.log('Sub 2:', value));

// Emit more values
hot$.next(3); // Both receive
hot$.next(4); // Both receive

// Output:
// Sub 1: 1
// Sub 1: 2
// Sub 1: 3
// Sub 2: 3  <- Joined late, gets values from this point
// Sub 1: 4
// Sub 2: 4
```

### DOM Events (Hot)

```typescript
import { fromEvent } from 'rxjs';

@Component({
  selector: 'app-click-tracker'
})
export class ClickTrackerComponent implements OnInit {
  // Hot - clicks happen regardless of subscriptions
  clicks$ = fromEvent(document, 'click');

  ngOnInit() {
    // First subscriber
    this.clicks$.subscribe(() => console.log('Subscriber 1: clicked'));

    setTimeout(() => {
      // Second subscriber joins 2 seconds later
      // Will only receive clicks from this point forward
      this.clicks$.subscribe(() => console.log('Subscriber 2: clicked'));
    }, 2000);

    // Both subscribers receive the same click events
    // They're listening to the same event source
  }
}
```

### WebSocket Connection (Hot)

```typescript
import { webSocket } from 'rxjs/webSocket';

@Injectable()
export class WebSocketService {
  // Hot - single WebSocket connection shared by all subscribers
  private socket$ = webSocket('wss://api.example.com/feed');

  getMessages(): Observable<any> {
    return this.socket$;
  }
}

@Component({
  selector: 'app-live-feed'
})
export class LiveFeedComponent implements OnInit {
  private ws = inject(WebSocketService);

  ngOnInit() {
    // First subscriber
    this.ws.getMessages().subscribe(msg => 
      console.log('Component 1:', msg)
    );

    // Second subscriber shares the same WebSocket connection
    this.ws.getMessages().subscribe(msg => 
      console.log('Component 2:', msg)
    );

    // Only ONE WebSocket connection is created
    // Both subscribers receive the same messages
  }
}
```

### Shared State with BehaviorSubject

```typescript
@Injectable()
export class CounterService {
  // Hot observable with current value
  private counterSubject = new BehaviorSubject<number>(0);
  counter$ = this.counterSubject.asObservable();

  increment(): void {
    this.counterSubject.next(this.counterSubject.value + 1);
  }

  decrement(): void {
    this.counterSubject.next(this.counterSubject.value - 1);
  }
}

// Component 1
@Component({
  selector: 'app-counter-display'
})
export class CounterDisplayComponent {
  private counter = inject(CounterService);
  count$ = this.counter.counter$; // Shares state with other components
}

// Component 2
@Component({
  selector: 'app-counter-controls'
})
export class CounterControlsComponent {
  private counter = inject(CounterService);
  count$ = this.counter.counter$; // Same shared state

  increment() {
    this.counter.increment(); // All subscribers see the update
  }
}
```

## Key Differences

### Comparison Table

| Aspect | Cold Observable | Hot Observable |
|--------|----------------|----------------|
| **Producer** | Created per subscriber | Shared among subscribers |
| **Start** | On subscription | Independent of subscriptions |
| **Values** | From beginning | From subscription point |
| **Execution** | Unicast (separate) | Multicast (shared) |
| **Examples** | HTTP, interval, range | Subjects, DOM events, WebSocket |

### Visual Comparison

```typescript
// COLD: Each subscriber gets own execution
const cold$ = new Observable(observer => {
  const value = Math.random();
  observer.next(value);
});

cold$.subscribe(v => console.log('A:', v)); // A: 0.123
cold$.subscribe(v => console.log('B:', v)); // B: 0.456 (different!)

// HOT: Subscribers share execution
const hot$ = new Subject();
hot$.subscribe(v => console.log('A:', v));
hot$.subscribe(v => console.log('B:', v));
hot$.next(Math.random()); 
// A: 0.789
// B: 0.789 (same!)
```

### Memory and Performance

```typescript
// COLD: Multiple executions = more memory/resources
const coldInterval$ = interval(1000);

coldInterval$.subscribe(); // Creates timer #1
coldInterval$.subscribe(); // Creates timer #2
coldInterval$.subscribe(); // Creates timer #3
// Three separate timers running!

// HOT: Single execution = efficient
const hotInterval$ = interval(1000).pipe(share());

hotInterval$.subscribe(); // Creates timer
hotInterval$.subscribe(); // Shares existing timer
hotInterval$.subscribe(); // Shares existing timer
// One timer shared by all!
```

## Converting Cold to Hot

### Using share()

```typescript
import { share } from 'rxjs/operators';

@Injectable()
export class DataService {
  constructor(private http: HttpClient) {}

  // Cold by default
  private coldData$ = this.http.get('/api/data');

  // Convert to hot - shared among subscribers
  hotData$ = this.coldData$.pipe(share());
}

// Only ONE HTTP request for multiple subscriptions
service.hotData$.subscribe(data => console.log('Sub 1:', data));
service.hotData$.subscribe(data => console.log('Sub 2:', data));
```

### Using shareReplay()

```typescript
import { shareReplay } from 'rxjs/operators';

@Injectable()
export class CachedDataService {
  constructor(private http: HttpClient) {}

  // Cold + cached
  data$ = this.http.get('/api/data').pipe(
    shareReplay(1) // Hot + caches last value
  );
}

// First subscriber triggers request
service.data$.subscribe(data => console.log('Sub 1:', data));

// Later subscribers get cached value immediately
setTimeout(() => {
  service.data$.subscribe(data => console.log('Sub 2:', data)); // Instant!
}, 5000);
```

### Using Subject as Bridge

```typescript
@Injectable()
export class StreamBridgeService {
  private coldSource$ = interval(1000);
  private hotSubject = new Subject<number>();

  constructor() {
    // Convert cold to hot by subscribing and forwarding
    this.coldSource$.subscribe(value => 
      this.hotSubject.next(value)
    );
  }

  getHotStream(): Observable<number> {
    return this.hotSubject.asObservable();
  }
}
```

### Using publish() and connect()

```typescript
import { publish, ConnectableObservable } from 'rxjs';

const cold$ = interval(1000);

// Convert to hot but don't start yet
const hot$ = cold$.pipe(publish()) as ConnectableObservable<number>;

// Add subscribers
hot$.subscribe(v => console.log('Sub 1:', v));
hot$.subscribe(v => console.log('Sub 2:', v));

// Manually start the shared execution
hot$.connect();
```

## When to Use Each

### Use Cold Observables When

1. **Independent data per subscriber needed**
```typescript
// Each user gets their own data
getUserData(userId: string): Observable<User> {
  return this.http.get<User>(`/api/users/${userId}`);
  // Cold - each call is independent
}
```

2. **Replay from start is desired**
```typescript
// Each animation starts from beginning
getAnimation(): Observable<number> {
  return interval(16).pipe(
    map(frame => /* animation calculation */)
  );
}
```

3. **On-demand computation**
```typescript
// Calculate only when needed
calculateExpensiveValue(): Observable<number> {
  return defer(() => of(expensiveCalculation()));
}
```

### Use Hot Observables When

1. **Sharing expensive resources**
```typescript
// Share WebSocket connection
@Injectable()
export class RealtimeService {
  private socket$ = webSocket('wss://api.example.com').pipe(
    share() // Hot - one connection for all
  );
}
```

2. **Synchronizing multiple subscribers**
```typescript
// All subscribers see same state updates
@Injectable()
export class StateService {
  private state = new BehaviorSubject<State>(initialState);
  state$ = this.state.asObservable(); // Hot - shared state
}
```

3. **Event streams**
```typescript
// Share DOM events
clicks$ = fromEvent(button, 'click').pipe(
  share() // Hot - one listener for all
);
```

4. **Preventing duplicate HTTP requests**
```typescript
// Share API response
config$ = this.http.get('/api/config').pipe(
  shareReplay(1) // Hot + cached
);
```

## Real-World Examples

### Example 1: Search with Shared Results

```typescript
@Injectable()
export class SearchService {
  constructor(private http: HttpClient) {}

  private searchCache = new Map<string, Observable<SearchResult[]>>();

  search(query: string): Observable<SearchResult[]> {
    if (!this.searchCache.has(query)) {
      // Convert cold HTTP to hot with caching
      const search$ = this.http
        .get<SearchResult[]>(`/api/search?q=${query}`)
        .pipe(
          shareReplay({
            bufferSize: 1,
            refCount: true,
            windowTime: 60000 // 1 minute cache
          })
        );

      this.searchCache.set(query, search$);
    }

    return this.searchCache.get(query)!;
  }
}
```

### Example 2: Live Price Updates

```typescript
@Injectable()
export class StockPriceService {
  // Hot - real-time updates shared by all subscribers
  private priceUpdates$ = new Subject<PriceUpdate>();

  constructor(private ws: WebSocketService) {
    // Connect to price feed
    this.ws.connect('/prices').subscribe(update => {
      this.priceUpdates$.next(update);
    });
  }

  getPriceUpdates(symbol: string): Observable<number> {
    return this.priceUpdates$.pipe(
      filter(update => update.symbol === symbol),
      map(update => update.price),
      distinctUntilChanged()
    );
  }
}

// Multiple components can subscribe without creating
// multiple WebSocket connections
@Component({
  selector: 'app-stock-ticker'
})
export class StockTickerComponent {
  private prices = inject(StockPriceService);

  applePrice$ = this.prices.getPriceUpdates('AAPL');
  googlePrice$ = this.prices.getPriceUpdates('GOOGL');
}
```

### Example 3: Authentication State

```typescript
@Injectable()
export class AuthService {
  // Hot - shared auth state across app
  private userSubject = new BehaviorSubject<User | null>(null);
  user$ = this.userSubject.asObservable();

  private tokenSubject = new BehaviorSubject<string | null>(null);
  token$ = this.tokenSubject.asObservable();

  // Derived hot observable
  isAuthenticated$ = this.user$.pipe(
    map(user => user !== null),
    distinctUntilChanged()
  );

  login(credentials: Credentials): Observable<User> {
    // Cold HTTP request
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => {
        // Update hot observables
        this.userSubject.next(user);
        this.tokenSubject.next(user.token);
      })
    );
  }

  logout(): void {
    this.userSubject.next(null);
    this.tokenSubject.next(null);
  }
}
```

### Example 4: Polling with Shared Results

```typescript
@Injectable()
export class ServerStatusService {
  constructor(private http: HttpClient) {}

  // Hot - shared polling across all components
  status$ = interval(5000).pipe(
    startWith(0),
    switchMap(() => this.http.get<ServerStatus>('/api/status')),
    shareReplay(1) // Share polling results
  );

  // All components see the same status updates
  // Only one poll happens every 5 seconds
}

@Component({
  selector: 'app-status-widget'
})
export class StatusWidgetComponent {
  private statusService = inject(ServerStatusService);
  status$ = this.statusService.status$; // Hot - shared
}
```

### Example 5: Form State Management

```typescript
@Injectable()
export class FormStateService {
  // Hot - shared form state
  private formData = new BehaviorSubject<FormData>({});
  formData$ = this.formData.asObservable();

  // Hot - validation status
  private isValid = new BehaviorSubject<boolean>(false);
  isValid$ = this.isValid.asObservable();

  updateField(field: string, value: any): void {
    const current = this.formData.value;
    this.formData.next({ ...current, [field]: value });
    this.validateForm();
  }

  private validateForm(): void {
    const data = this.formData.value;
    const valid = /* validation logic */;
    this.isValid.next(valid);
  }
}

// Multiple components can react to form changes
@Component({ selector: 'app-form-field' })
export class FormFieldComponent {
  private formState = inject(FormStateService);
  formData$ = this.formState.formData$; // Hot - shared
}

@Component({ selector: 'app-form-submit' })
export class FormSubmitComponent {
  private formState = inject(FormStateService);
  canSubmit$ = this.formState.isValid$; // Hot - shared
}
```

## Common Mistakes

### 1. Not Recognizing Cold Observables

```typescript
// WRONG: Assuming shared execution
const data$ = this.http.get('/api/data');
data$.subscribe(/* ... */); // Request #1
data$.subscribe(/* ... */); // Request #2 (duplicate!)

// RIGHT: Make it hot if sharing needed
const data$ = this.http.get('/api/data').pipe(shareReplay(1));
data$.subscribe(/* ... */); // Request #1
data$.subscribe(/* ... */); // Uses cached result
```

### 2. Making Everything Hot

```typescript
// WRONG: Unnecessary hot conversion
getUserById(id: string): Observable<User> {
  return this.http.get<User>(`/api/users/${id}`).pipe(
    share() // Why? Each call should be independent
  );
}

// RIGHT: Keep it cold for per-call independence
getUserById(id: string): Observable<User> {
  return this.http.get<User>(`/api/users/${id}`);
}
```

### 3. Memory Leaks with Hot Observables

```typescript
// WRONG: Never unsubscribes from hot observable
ngOnInit() {
  this.realtimeService.updates$.subscribe(/* ... */);
  // Component destroyed but subscription lives on
}

// RIGHT: Clean up subscriptions
ngOnInit() {
  this.realtimeService.updates$.pipe(
    takeUntilDestroyed()
  ).subscribe(/* ... */);
}
```

### 4. Expecting Replay on Hot Observables

```typescript
// WRONG: Expecting to receive previous values
const hot$ = new Subject();
hot$.next(1);
hot$.next(2);
hot$.subscribe(v => console.log(v)); // Won't receive 1 or 2
hot$.next(3); // Will receive 3

// RIGHT: Use BehaviorSubject or ReplaySubject if replay needed
const hot$ = new BehaviorSubject(0);
hot$.next(1);
hot$.next(2);
hot$.subscribe(v => console.log(v)); // Receives 2 immediately
```

## Best Practices

### 1. Document Hot vs Cold

```typescript
@Injectable()
export class DataService {
  /**
   * Gets user data.
   * COLD: Each subscription creates a new HTTP request.
   */
  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }

  /**
   * Streams real-time updates.
   * HOT: All subscriptions share the same WebSocket connection.
   */
  liveUpdates$: Observable<Update>;
}
```

### 2. Choose Appropriate Subject Type

```typescript
// Subject - no initial value, no replay
const events$ = new Subject<Event>();

// BehaviorSubject - has current value
const state$ = new BehaviorSubject<State>(initialState);

// ReplaySubject - replays N previous values
const history$ = new ReplaySubject<Action>(10);

// AsyncSubject - emits last value on complete
const result$ = new AsyncSubject<Result>();
```

### 3. Convert Cold to Hot Strategically

```typescript
@Injectable()
export class StrategicService {
  // Keep cold - each call should be independent
  getById(id: string): Observable<Data> {
    return this.http.get(`/api/data/${id}`);
  }

  // Make hot - expensive, should be shared
  getAllData(): Observable<Data[]> {
    return this.http.get<Data[]>('/api/data').pipe(
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }
}
```

### 4. Use refCount for Cleanup

```typescript
// Automatically unsubscribes when no subscribers
const data$ = this.http.get('/api/data').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

## Interview Questions

### Q1: What's the main difference between hot and cold observables?

**Answer:** Cold observables create a new producer for each subscriber (unicast), starting from the beginning. Hot observables share one producer among all subscribers (multicast), with new subscribers joining in progress. Example: HTTP requests are cold (separate request per subscription), DOM events are hot (shared event listener).

### Q2: How do you convert a cold observable to hot?

**Answer:** Use operators like `share()`, `shareReplay()`, or manually with `publish()` + `connect()`. Or use a Subject to bridge: subscribe to the cold observable and forward emissions to a Subject.

```typescript
// Using share
const hot$ = cold$.pipe(share());

// Using Subject bridge
const subject = new Subject();
cold$.subscribe(v => subject.next(v));
```

### Q3: Why are HTTP requests cold observables?

**Answer:** Each subscription to an HTTP observable creates a new HTTP request. This is intentional - you want independent requests that can be cancelled, retried, or modified per subscription. If you need to share one request among multiple subscribers, use `shareReplay()`.

### Q4: When would you use a BehaviorSubject over a Subject?

**Answer:** Use BehaviorSubject when you need:
1. An initial/current value
2. New subscribers to immediately receive the latest value
3. Synchronous access to current value via `.value`

Example: Current user state, app configuration, or any "current state" scenario.

## Key Takeaways

1. **Cold** = unicast, new producer per subscriber, starts from beginning
2. **Hot** = multicast, shared producer, join in progress
3. HTTP requests and intervals are **cold** by default
4. DOM events, Subjects, and WebSockets are **hot**
5. Use `share()` for multicasting without caching
6. Use `shareReplay()` for multicasting with caching
7. Cold is default and appropriate for most cases
8. Convert to hot only when sharing is beneficial
9. Hot observables need subscription cleanup
10. Document hot/cold behavior in your services

## Resources

- [Ben Lesh - Hot vs Cold Observables](https://medium.com/@benlesh/hot-vs-cold-observables-f8094ed53339)
- [RxJS Documentation - Hot vs Cold](https://rxjs.dev/guide/glossary-and-semantics#hot-and-cold-observables)
- [Learn RxJS - Understanding Observables](https://www.learnrxjs.io/learn-rxjs/concepts/hot-vs-cold-observables)
- [Egghead.io - RxJS Beyond the Basics: Creating Hot Observables](https://egghead.io/courses/rxjs-beyond-the-basics-creating-observables-from-scratch)
