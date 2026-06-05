# Zoneless Angular: Signal-Based Change Detection

## The Idea

**In plain English:** Zoneless Angular is a way to build Angular apps where the app only updates the parts of the screen that actually changed, instead of constantly checking everything. A "signal" is a special container that holds a value and automatically tells the app when that value changes.

**Real-world analogy:** Imagine a large office building where every room has a light switch. In the old system, a security guard walks the entire building every few minutes checking every single room — even empty ones. In the new (zoneless) system, each room has a smart sensor that sends an alert directly to maintenance only when something in that specific room changes.

- The security guard walking every room = Zone.js triggering change detection across the whole app
- Each smart sensor in a room = a signal attached to a specific piece of data
- Maintenance only going to the room that sent the alert = Angular updating only the components whose signals changed

---

## Overview

Zoneless Angular represents a fundamental shift in how Angular detects and propagates changes throughout an application. By eliminating the dependency on Zone.js and embracing signals as the primary reactivity primitive, Angular achieves better performance, simpler mental models, and more predictable behavior.

This transition from Zone.js-based automatic change detection to signal-based reactivity is one of the most significant architectural changes in Angular's history, affecting how developers write components, manage state, and optimize performance.

## The Problem with Zone.js

### Automatic but Unpredictable

Zone.js patches all async APIs (setTimeout, Promise, XMLHttpRequest, etc.) to automatically trigger change detection. While convenient, this approach has significant drawbacks:

```typescript
// With Zone.js - change detection runs after EVERY async operation
@Component({
  selector: 'app-user-profile',
  template: `
    <div>{{ userData?.name }}</div>
    <div>Last updated: {{ lastUpdate }}</div>
  `
})
export class UserProfileComponent {
  userData: User | null = null;
  lastUpdate = new Date();

  ngOnInit() {
    // Change detection runs after this
    setTimeout(() => {
      console.log('Timer fired');
    }, 1000);

    // Change detection runs after this
    fetch('/api/user')
      .then(res => res.json())
      .then(data => {
        this.userData = data; // Actually needs update
      });

    // Change detection runs after this too
    Promise.resolve().then(() => {
      // No template changes, but CD still runs
      this.logAnalytics();
    });
  }

  private logAnalytics() {
    // Side effect that doesn't affect the view
    console.log('Analytics logged');
  }
}
```

Problems:
1. **Over-triggering**: Change detection runs even when nothing in the view changed
2. **Unpredictable timing**: Hard to know when CD will run
3. **Performance overhead**: Zone.js patches add runtime cost
4. **Bundle size**: Zone.js adds ~12-15KB to the bundle
5. **Third-party conflicts**: Zone.js patches can interfere with libraries

### Performance Implications

```typescript
// Zone.js triggers change detection for everything
@Component({
  selector: 'app-dashboard',
  template: `
    <div *ngFor="let item of items">
      {{ item.name }}
    </div>
    <button (click)="refresh()">Refresh</button>
  `
})
export class DashboardComponent {
  items: Item[] = [];

  constructor(private http: HttpClient) {}

  refresh() {
    // Zone.js intercepts this
    this.http.get('/api/items').subscribe(items => {
      this.items = items;
      // Change detection runs for ENTIRE application tree
      // Not just this component
    });

    // Even unrelated timers trigger CD
    setInterval(() => {
      this.checkServerStatus(); // No view changes
    }, 5000);
  }

  private checkServerStatus() {
    // Background task, but still triggers CD
    fetch('/api/health').then(/* ... */);
  }
}
```

## Zoneless Architecture

### Signal-Based Reactivity

Zoneless Angular uses signals to create a dependency graph that precisely tracks what needs to update:

```typescript
// Zoneless approach with signals
@Component({
  selector: 'app-user-profile',
  template: `
    <div>{{ userData()?.name }}</div>
    <div>Last updated: {{ lastUpdate() }}</div>
    <div>Full name: {{ fullName() }}</div>
  `,
  standalone: true
})
export class UserProfileComponent {
  // Signals create reactive dependencies
  userData = signal<User | null>(null);
  lastUpdate = signal(new Date());

  // Computed signals automatically track dependencies
  fullName = computed(() => {
    const user = this.userData();
    return user ? `${user.firstName} ${user.lastName}` : 'Anonymous';
  });

  constructor(private http: HttpClient) {
    // Only updates when signal changes
    effect(() => {
      console.log('User changed:', this.userData());
    });
  }

  ngOnInit() {
    // Regular timeout - no automatic change detection
    setTimeout(() => {
      console.log('Timer fired');
      // View doesn't update unless we update a signal
    }, 1000);

    // Fetch data without Zone.js
    fetch('/api/user')
      .then(res => res.json())
      .then(data => {
        // Explicitly update signal - only this triggers update
        this.userData.set(data);
        this.lastUpdate.set(new Date());
      });
  }
}
```

### How It Works

```typescript
// Signal implementation (simplified)
class Signal<T> {
  private value: T;
  private subscribers = new Set<Subscriber>();
  private dependencies = new Set<Signal<any>>();

  constructor(initialValue: T) {
    this.value = initialValue;
  }

  // Reading a signal registers dependency
  get(): T {
    if (currentComputation) {
      this.subscribers.add(currentComputation);
      currentComputation.dependencies.add(this);
    }
    return this.value;
  }

  // Writing a signal notifies subscribers
  set(newValue: T): void {
    if (this.value !== newValue) {
      this.value = newValue;
      this.notifySubscribers();
    }
  }

  update(updater: (current: T) => T): void {
    this.set(updater(this.value));
  }

  private notifySubscribers(): void {
    // Only notify what actually depends on this signal
    this.subscribers.forEach(subscriber => {
      subscriber.notify();
    });
  }
}

// Computed signal tracks dependencies automatically
function computed<T>(computation: () => T): Signal<T> {
  const signal = new Signal<T>(undefined as any);
  
  const recompute = () => {
    const prevComputation = currentComputation;
    currentComputation = signal;
    
    try {
      // Run computation, tracking dependencies
      const value = computation();
      signal.set(value);
    } finally {
      currentComputation = prevComputation;
    }
  };

  recompute(); // Initial computation
  return signal;
}
```

## Enabling Zoneless Mode

### Configuration

```typescript
// main.ts - Enable zoneless mode
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    // Enable zoneless change detection
    provideExperimentalZonelessChangeDetection(),
    // Other providers
  ]
}).catch(err => console.error(err));
```

### Migration Strategy

```typescript
// Step 1: Identify Zone.js dependencies
// Before - relies on Zone.js
@Component({
  selector: 'app-counter',
  template: `<div>Count: {{ count }}</div>`
})
export class CounterComponent {
  count = 0;

  increment() {
    setTimeout(() => {
      this.count++; // Zone.js triggers CD
    }, 100);
  }
}

// After - explicit with signals
@Component({
  selector: 'app-counter',
  template: `<div>Count: {{ count() }}</div>`,
  standalone: true
})
export class CounterComponent {
  count = signal(0);

  increment() {
    setTimeout(() => {
      this.count.update(c => c + 1); // Explicit update
    }, 100);
  }
}
```

### Handling Third-Party Libraries

```typescript
// Wrapper for libraries that expect Zone.js
@Injectable()
export class ThirdPartyService {
  private data = signal<any>(null);

  constructor() {
    // Library callback won't trigger CD automatically
    someLibrary.onData((newData) => {
      // Manually update signal to trigger reactivity
      this.data.set(newData);
    });
  }

  getData() {
    return this.data.asReadonly();
  }
}

// Alternative: Use ChangeDetectorRef for legacy code
@Component({
  selector: 'app-legacy',
  template: `<div>{{ data }}</div>`
})
export class LegacyComponent {
  data: any;

  constructor(private cdr: ChangeDetectorRef) {
    someLibrary.onData((newData) => {
      this.data = newData;
      // Manually trigger change detection
      this.cdr.markForCheck();
    });
  }
}
```

## Signal Patterns in Zoneless Angular

### State Management

```typescript
// Reactive state store with signals
@Injectable({ providedIn: 'root' })
export class UserStore {
  // Private writable signals
  private _users = signal<User[]>([]);
  private _loading = signal(false);
  private _error = signal<string | null>(null);

  // Public readonly signals
  readonly users = this._users.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error = this._error.asReadonly();

  // Computed derived state
  readonly userCount = computed(() => this._users().length);
  readonly hasUsers = computed(() => this._users().length > 0);
  readonly sortedUsers = computed(() => 
    [...this._users()].sort((a, b) => a.name.localeCompare(b.name))
  );

  constructor(private http: HttpClient) {}

  async loadUsers() {
    this._loading.set(true);
    this._error.set(null);

    try {
      const users = await fetch('/api/users').then(r => r.json());
      this._users.set(users);
    } catch (err) {
      this._error.set(err.message);
    } finally {
      this._loading.set(false);
    }
  }

  addUser(user: User) {
    this._users.update(users => [...users, user]);
  }

  removeUser(id: string) {
    this._users.update(users => users.filter(u => u.id !== id));
  }

  updateUser(id: string, updates: Partial<User>) {
    this._users.update(users =>
      users.map(u => u.id === id ? { ...u, ...updates } : u)
    );
  }
}

// Component using the store
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="store.loading()">Loading...</div>
    <div *ngIf="store.error()">Error: {{ store.error() }}</div>
    
    <div *ngIf="!store.loading() && store.hasUsers()">
      <p>Total users: {{ store.userCount() }}</p>
      <ul>
        <li *ngFor="let user of store.sortedUsers()">
          {{ user.name }}
          <button (click)="removeUser(user.id)">Remove</button>
        </li>
      </ul>
    </div>
    
    <button (click)="store.loadUsers()">Refresh</button>
  `,
  standalone: true
})
export class UserListComponent {
  constructor(public store: UserStore) {}

  removeUser(id: string) {
    if (confirm('Delete user?')) {
      this.store.removeUser(id);
    }
  }
}
```

### Effect Patterns

```typescript
// Effects for side effects
@Component({
  selector: 'app-search',
  template: `
    <input [value]="query()" (input)="query.set($event.target.value)">
    <div *ngIf="searching()">Searching...</div>
    <ul>
      <li *ngFor="let result of results()">{{ result.title }}</li>
    </ul>
  `,
  standalone: true
})
export class SearchComponent {
  query = signal('');
  results = signal<SearchResult[]>([]);
  searching = signal(false);

  constructor() {
    // Effect runs when query changes
    effect(() => {
      const query = this.query();
      
      if (query.length < 3) {
        this.results.set([]);
        return;
      }

      this.performSearch(query);
    });

    // Cleanup effect
    effect((onCleanup) => {
      const subscription = this.searchService.subscribe(
        results => this.results.set(results)
      );

      onCleanup(() => {
        subscription.unsubscribe();
      });
    });
  }

  private searchDebouncer = new Subject<string>();

  private performSearch(query: string) {
    this.searching.set(true);
    
    fetch(`/api/search?q=${encodeURIComponent(query)}`)
      .then(r => r.json())
      .then(results => {
        this.results.set(results);
        this.searching.set(false);
      });
  }
}
```

### Signal-Based Forms

```typescript
// Reactive forms with signals
@Component({
  selector: 'app-user-form',
  template: `
    <form (submit)="handleSubmit($event)">
      <input 
        [value]="name()" 
        (input)="name.set($event.target.value)"
        placeholder="Name">
      
      <input 
        [value]="email()" 
        (input)="email.set($event.target.value)"
        placeholder="Email"
        type="email">
      
      <div *ngIf="emailError()">{{ emailError() }}</div>
      
      <button [disabled]="!isValid()">Submit</button>
    </form>
    
    <div *ngIf="formData()">
      <h3>Form Data:</h3>
      <pre>{{ formData() | json }}</pre>
    </div>
  `,
  standalone: true
})
export class UserFormComponent {
  name = signal('');
  email = signal('');

  // Computed validation
  emailError = computed(() => {
    const email = this.email();
    if (!email) return null;
    
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email) ? null : 'Invalid email format';
  });

  isValid = computed(() => {
    return this.name().length > 0 && 
           this.email().length > 0 && 
           !this.emailError();
  });

  formData = signal<any>(null);

  constructor() {
    // Log form changes
    effect(() => {
      console.log('Form state:', {
        name: this.name(),
        email: this.email(),
        valid: this.isValid()
      });
    });
  }

  handleSubmit(event: Event) {
    event.preventDefault();
    
    if (this.isValid()) {
      this.formData.set({
        name: this.name(),
        email: this.email(),
        submittedAt: new Date()
      });

      // Reset form
      this.name.set('');
      this.email.set('');
    }
  }
}
```

## Performance Optimizations

### Selective Updates

```typescript
// Only affected components update
@Component({
  selector: 'app-parent',
  template: `
    <app-child-a [data]="dataA()"></app-child-a>
    <app-child-b [data]="dataB()"></app-child-b>
    <app-child-c [data]="dataC()"></app-child-c>
  `,
  standalone: true
})
export class ParentComponent {
  dataA = signal({ value: 1 });
  dataB = signal({ value: 2 });
  dataC = signal({ value: 3 });

  updateA() {
    // Only ChildA updates, not ChildB or ChildC
    this.dataA.update(d => ({ value: d.value + 1 }));
  }

  updateB() {
    // Only ChildB updates
    this.dataB.update(d => ({ value: d.value + 1 }));
  }
}
```

### Batched Updates

```typescript
// Signal updates are automatically batched
@Component({
  selector: 'app-batch-demo',
  template: `
    <div>Count: {{ count() }}</div>
    <div>Doubled: {{ doubled() }}</div>
    <div>Status: {{ status() }}</div>
  `,
  standalone: true
})
export class BatchDemoComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);
  status = signal('idle');

  updateMultiple() {
    // These updates are batched into a single change detection cycle
    this.count.set(10);
    this.count.set(20);
    this.count.set(30); // Only this final value triggers update
    this.status.set('updated');
    
    // Only ONE template update happens, not three
  }

  // Manual batching with untracked
  updateWithSideEffect() {
    this.count.update(c => {
      const newValue = c + 1;
      
      // Side effect that shouldn't trigger tracking
      untracked(() => {
        console.log('Current:', c, 'New:', newValue);
        this.logToAnalytics(newValue);
      });
      
      return newValue;
    });
  }

  private logToAnalytics(value: number) {
    // Analytics logging without triggering reactivity
  }
}
```

### Memory Efficiency

```typescript
// Signals are more memory efficient than Zone.js
@Component({
  selector: 'app-large-list',
  template: `
    <div *ngFor="let item of items()">
      {{ item.name }} - {{ item.status() }}
    </div>
  `,
  standalone: true
})
export class LargeListComponent {
  // Each item has its own signal
  items = signal<ItemWithSignal[]>([]);

  constructor() {
    this.loadItems();
  }

  loadItems() {
    fetch('/api/items').then(r => r.json()).then(data => {
      // Create items with individual signals
      const items = data.map(item => ({
        ...item,
        status: signal(item.status)
      }));
      this.items.set(items);
    });
  }

  updateItemStatus(index: number, newStatus: string) {
    // Only the specific item updates, not entire list
    const item = this.items()[index];
    item.status.set(newStatus);
  }
}
```

## Integration with RxJS

### Signal-Observable Interop

```typescript
// Converting between signals and observables
@Component({
  selector: 'app-interop',
  template: `
    <div>Signal value: {{ signalValue() }}</div>
    <div>Observable value: {{ observableValue() }}</div>
  `,
  standalone: true
})
export class InteropComponent {
  // Signal to Observable
  signalValue = signal(0);
  signalAsObservable$ = toObservable(this.signalValue);

  // Observable to Signal
  private interval$ = interval(1000);
  observableValue = toSignal(this.interval$, { initialValue: 0 });

  constructor() {
    // Subscribe to signal changes as observable
    this.signalAsObservable$.subscribe(value => {
      console.log('Signal changed:', value);
    });

    // Effect watches signal from observable
    effect(() => {
      console.log('Observable value:', this.observableValue());
    });
  }

  incrementSignal() {
    this.signalValue.update(v => v + 1);
  }
}
```

### Complex Async Patterns

```typescript
// Combining signals with async operations
@Component({
  selector: 'app-data-loader',
  standalone: true
})
export class DataLoaderComponent {
  private userId = signal<string>('');
  
  // Automatically refetch when userId changes
  private userData = toSignal(
    toObservable(this.userId).pipe(
      filter(id => id.length > 0),
      debounceTime(300),
      switchMap(id => this.http.get(`/api/users/${id}`))
    ),
    { initialValue: null }
  );

  constructor(private http: HttpClient) {
    effect(() => {
      const user = this.userData();
      if (user) {
        console.log('User loaded:', user);
      }
    });
  }

  setUserId(id: string) {
    this.userId.set(id);
    // userData automatically updates via RxJS pipeline
  }
}
```

## Common Misconceptions

### "Zoneless means no automatic change detection"

**Reality**: Zoneless Angular still has automatic change detection, but it's precise and predictable. Signals automatically track dependencies and trigger updates only where needed.

```typescript
// This STILL updates automatically
@Component({
  template: `<div>{{ count() }}</div>`
})
export class Component {
  count = signal(0);
  
  increment() {
    this.count.update(c => c + 1); // Automatically updates template
  }
}
```

### "You must convert everything to signals immediately"

**Reality**: Migration can be gradual. You can mix signals with traditional change detection:

```typescript
@Component({
  template: `
    <div>Signal: {{ signalData() }}</div>
    <div>Traditional: {{ traditionalData }}</div>
  `
})
export class MixedComponent {
  signalData = signal('signal');
  traditionalData = 'traditional';
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  update() {
    this.signalData.set('updated'); // Automatic
    this.traditionalData = 'updated'; // Need manual CD
    this.cdr.markForCheck();
  }
}
```

### "Effects replace lifecycle hooks"

**Reality**: Effects complement lifecycle hooks, they don't replace them:

```typescript
@Component({})
export class Component {
  data = signal<Data | null>(null);
  
  // Still use lifecycle hooks
  ngOnInit() {
    this.loadData();
  }
  
  // Effects for reactive side effects
  constructor() {
    effect(() => {
      const data = this.data();
      if (data) {
        this.processData(data);
      }
    });
  }
}
```

## Performance Implications

### Benefits

1. **Smaller bundles**: Removing Zone.js saves 12-15KB
2. **Faster runtime**: No async API patching overhead
3. **Predictable performance**: Updates only what changed
4. **Better tree-shaking**: Unused signals aren't included
5. **Improved debugging**: Clear dependency graph

### Benchmarks

```typescript
// Zone.js approach
// 1000 components, 1 property change: ~15-20ms
// Change detection checks entire tree

// Zoneless approach
// 1000 components, 1 signal change: ~2-3ms
// Only affected components update
```

### Optimization Strategies

```typescript
// 1. Use computed for derived state
const fullName = computed(() => `${firstName()} ${lastName()}`);
// Not: const fullName = () => `${firstName()} ${lastName()}`

// 2. Batch updates
untracked(() => {
  signal1.set(value1);
  signal2.set(value2);
  signal3.set(value3);
});

// 3. Use readonly signals for public APIs
private _data = signal<Data>([]);
readonly data = this._data.asReadonly();

// 4. Avoid unnecessary effects
// Bad: effect(() => console.log(signal())); // Logs on every change
// Good: Use sparingly for true side effects
```

## Interview Questions

### Q1: How does zoneless Angular differ from Zone.js-based change detection?

**Answer**: Zoneless Angular uses signals for reactive change detection instead of Zone.js monkey-patching. Zone.js automatically patches all async APIs (setTimeout, Promise, fetch, etc.) and triggers change detection after any async operation completes, checking the entire component tree. This is convenient but leads to over-checking and unpredictable performance.

Zoneless Angular with signals creates an explicit dependency graph. When you read a signal in a template or computed, Angular tracks that dependency. When the signal updates, only the specific components/computations that depend on it re-run. This provides precise, predictable updates with better performance and smaller bundle sizes (Zone.js is ~12-15KB).

### Q2: How do you migrate an existing Angular app to zoneless mode?

**Answer**: Migration strategy:
1. Enable experimental zoneless mode with `provideExperimentalZonelessChangeDetection()`
2. Convert component properties to signals incrementally
3. Replace properties used in templates with signal reads: `{{ count }}` becomes `{{ count() }}`
4. Update event handlers to use signal setters: `this.count = 5` becomes `this.count.set(5)`
5. For third-party libraries expecting Zone.js, manually trigger updates using `ChangeDetectorRef.markForCheck()` or wrap callbacks to update signals
6. Use `toSignal()` and `toObservable()` for RxJS interop
7. Test thoroughly as async operations won't automatically trigger change detection

You can migrate gradually - signals and traditional change detection can coexist.

### Q3: What are signals and how do they work internally?

**Answer**: Signals are reactive primitives that hold values and track dependencies. Internally, a signal maintains:
- The current value
- A set of subscribers (computations/effects that read it)
- Dependency tracking during reads

When you read a signal (call it as a function), it registers the current computation as a subscriber. When you write to a signal (`set()` or `update()`), it notifies all subscribers to re-run.

Computed signals automatically track dependencies by recording which signals are read during their computation. They lazily recompute only when dependencies change and someone reads them. Effects are similar but run eagerly for side effects. This creates a directed acyclic graph (DAG) of dependencies, enabling precise updates - only affected parts of the application re-render when a signal changes.

### Q4: What are effects and when should you use them?

**Answer**: Effects are functions that run reactively when their dependencies (signals they read) change. They're used for side effects like logging, analytics, localStorage sync, or DOM manipulation:

```typescript
effect(() => {
  const user = userSignal();
  console.log('User changed:', user);
  localStorage.setItem('user', JSON.stringify(user));
});
```

Use effects for:
- Synchronizing with external systems
- Logging/debugging
- Managing subscriptions
- Side effects that don't produce values

Avoid using effects for:
- Deriving values (use `computed()` instead)
- Propagating signal values (signals do this automatically)
- Everything (overuse makes code hard to follow)

Effects can return cleanup functions for resource disposal, and you can use `untracked()` to read signals without creating dependencies.

### Q5: How do signals integrate with RxJS?

**Answer**: Angular provides interop utilities:

`toObservable()` converts a signal to an Observable that emits when the signal changes:
```typescript
const count = signal(0);
const count$ = toObservable(count);
```

`toSignal()` converts an Observable to a signal:
```typescript
const data$ = http.get('/api/data');
const data = toSignal(data$, { initialValue: null });
```

This enables powerful patterns like:
```typescript
const userId = signal('123');
const userData = toSignal(
  toObservable(userId).pipe(
    switchMap(id => http.get(`/api/users/${id}`))
  )
);
```

When userId changes, the HTTP request automatically re-fires and userData updates. This combines signal reactivity with RxJS operators for async operations.

### Q6: What are computed signals and how do they differ from regular signals?

**Answer**: Computed signals derive values from other signals and are memoized/lazy:

```typescript
const firstName = signal('John');
const lastName = signal('Doe');
const fullName = computed(() => `${firstName()} ${lastName()}`);
```

Key differences:
1. **Read-only**: Can't call `set()` on computed signals
2. **Lazy**: Only recompute when read and dependencies changed
3. **Memoized**: Cache result until dependencies change
4. **Automatic dependencies**: Track which signals they read

Regular signals are writable and eagerly store values. Computed signals are more efficient for derived state - if nothing reads the computed, it never runs. They also prevent infinite loops by detecting circular dependencies. Use computed for any value derived from other signals, use regular signals for source state.

### Q7: How does batching work in zoneless Angular?

**Answer**: Signal updates are automatically batched within a single synchronous execution context. Multiple signal updates trigger only one change detection cycle:

```typescript
function updateMultiple() {
  count.set(1);
  count.set(2);
  count.set(3);
  status.set('done');
  // Only ONE template update occurs
}
```

Angular queues all signal notifications and flushes them together. This happens automatically - you don't need to do anything special.

For more control, use `untracked()` to prevent creating dependencies or triggering updates:
```typescript
effect(() => {
  const value = signal();
  untracked(() => {
    // Reads here don't create dependencies
    console.log(otherSignal());
  });
});
```

Batching significantly improves performance when making multiple related updates, preventing unnecessary intermediate renders.

### Q8: What are the main performance benefits of zoneless Angular?

**Answer**: 
1. **Smaller bundles**: Removing Zone.js saves 12-15KB (and its transitive dependencies)
2. **No monkey-patching overhead**: Zone.js patches all async APIs at runtime, adding performance cost to every async operation
3. **Precise updates**: Only components depending on changed signals update, not the entire tree
4. **Predictable performance**: No hidden change detection cycles from third-party code or timers
5. **Better tree-shaking**: Unused signals/computeds are removed from builds
6. **Faster async operations**: No Zone.js interception of promises, timers, fetch, etc.
7. **Reduced memory usage**: Smaller change detection structures, no zone context tracking

Benchmarks show 3-5x faster change detection in apps with many components. Real-world apps see 20-40% improvement in runtime performance and improved Core Web Vitals scores.

## Key Takeaways

1. Zoneless Angular uses signals for precise, predictable change detection without Zone.js
2. Signals create an explicit dependency graph, updating only affected components
3. Migration can be gradual - signals coexist with traditional change detection
4. Use `signal()` for writable state, `computed()` for derived values, `effect()` for side effects
5. Signals integrate with RxJS via `toSignal()` and `toObservable()`
6. Performance improvements include smaller bundles, faster updates, and predictable behavior
7. Effects should be used sparingly for true side effects, not value propagation
8. Batching happens automatically - multiple updates trigger one change detection cycle

## Resources

- [Angular Signals RFC](https://github.com/angular/angular/discussions/49685)
- [Angular Zoneless Documentation](https://angular.dev/guide/experimental/zoneless)
- [Signal-based Components Guide](https://angular.dev/guide/signals)
- [Migrating to Zoneless Angular](https://blog.angular.io/moving-to-zoneless-angular)
- [Understanding Angular Signals](https://www.youtube.com/watch?v=signalstalk)
- [Signals Deep Dive by Alex Rickabaugh](https://www.youtube.com/watch?v=deepdive)
- [RxJS and Signals Interop](https://angular.dev/guide/signals/rxjs-interop)
