# RxJS Creation Operators

## Table of Contents
- [Introduction](#introduction)
- [of Operator](#of-operator)
- [from Operator](#from-operator)
- [fromEvent Operator](#fromevent-operator)
- [interval and timer](#interval-and-timer)
- [defer Operator](#defer-operator)
- [range and generate](#range-and-generate)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Creation operators are functions that create new Observables from various sources. They're the starting point for most reactive streams in Angular applications.

Understanding creation operators is essential for:
- Converting different data sources to Observables
- Creating test data streams
- Handling events and timers
- Building reactive workflows

## of Operator

`of` creates an Observable that emits values in sequence and completes:

```typescript
import { of } from 'rxjs';

// Emit multiple values
const numbers$ = of(1, 2, 3, 4, 5);

numbers$.subscribe({
  next: (value) => console.log('Value:', value),
  complete: () => console.log('Complete')
});

// Output:
// Value: 1
// Value: 2
// Value: 3
// Value: 4
// Value: 5
// Complete
```

### of with Different Types

```typescript
import { of } from 'rxjs';

// Numbers
const nums$ = of(1, 2, 3);

// Strings
const strings$ = of('hello', 'world');

// Objects
const users$ = of(
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
);

// Mixed types
const mixed$ = of(42, 'text', true, { key: 'value' });

// Single value
const single$ = of('single value');

// Empty (completes immediately)
const empty$ = of();
```

### of in Angular Services

```typescript
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Injectable({
  providedIn: 'root'
})
export class MockProductService {
  private products: Product[] = [
    { id: 1, name: 'Product 1', price: 99 },
    { id: 2, name: 'Product 2', price: 149 },
    { id: 3, name: 'Product 3', price: 199 }
  ];
  
  // Return mock data as Observable
  getProducts(): Observable<Product[]> {
    return of(this.products).pipe(
      delay(500) // Simulate network delay
    );
  }
  
  getProductById(id: number): Observable<Product | undefined> {
    const product = this.products.find(p => p.id === id);
    return of(product);
  }
  
  // Mock error for testing
  getProductsWithError(): Observable<Product[]> {
    return of(this.products).pipe(
      delay(500),
      tap(() => {
        throw new Error('Mock error');
      })
    );
  }
}
```

### of for Synchronous Values

```typescript
import { Component, OnInit } from '@angular/core';
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-of-demo',
  template: `
    <div>{{ result$ | async }}</div>
  `
})
export class OfDemoComponent {
  // Convert synchronous data to Observable
  result$ = of('Hello', 'World').pipe(
    map(words => words.join(' '))
  );
  
  // Conditional Observable
  getGreeting(hour: number) {
    if (hour < 12) {
      return of('Good morning');
    } else if (hour < 18) {
      return of('Good afternoon');
    } else {
      return of('Good evening');
    }
  }
}
```

## from Operator

`from` converts arrays, promises, iterables, or observables to Observable:

```typescript
import { from } from 'rxjs';

// From array
const fromArray$ = from([1, 2, 3, 4, 5]);
fromArray$.subscribe(val => console.log('Array:', val));
// Emits: 1, 2, 3, 4, 5

// From string (iterable)
const fromString$ = from('hello');
fromString$.subscribe(char => console.log('Char:', char));
// Emits: 'h', 'e', 'l', 'l', 'o'

// From Set
const fromSet$ = from(new Set([1, 2, 3, 2, 1]));
fromSet$.subscribe(val => console.log('Set:', val));
// Emits: 1, 2, 3

// From Map
const map = new Map([
  ['key1', 'value1'],
  ['key2', 'value2']
]);
const fromMap$ = from(map);
fromMap$.subscribe(([key, value]) => console.log(key, ':', value));
```

### from with Promises

```typescript
import { from } from 'rxjs';
import { catchError } from 'rxjs/operators';

// Convert Promise to Observable
const promise = fetch('/api/data').then(r => r.json());
const observable$ = from(promise);

observable$.subscribe({
  next: (data) => console.log('Data:', data),
  error: (err) => console.error('Error:', err)
});

// In Angular component
@Component({
  selector: 'app-promise-demo',
  template: `
    <div *ngIf="data$ | async as data">
      {{ data | json }}
    </div>
  `
})
export class PromiseDemoComponent {
  data$ = from(
    fetch('/api/user').then(r => r.json())
  ).pipe(
    catchError(err => of({ error: err.message }))
  );
}
```

### from with Async Iterators

```typescript
import { from } from 'rxjs';

// Async generator function
async function* asyncGenerator() {
  yield 1;
  await new Promise(resolve => setTimeout(resolve, 1000));
  yield 2;
  await new Promise(resolve => setTimeout(resolve, 1000));
  yield 3;
}

// Convert to Observable
const asyncIterable$ = from(asyncGenerator());

asyncIterable$.subscribe({
  next: (value) => console.log('Value:', value),
  complete: () => console.log('Complete')
});
// Emits: 1 (after 0s), 2 (after 1s), 3 (after 2s)
```

### from in Angular Services

```typescript
import { Injectable } from '@angular/core';
import { from, Observable } from 'rxjs';
import { mergeMap, toArray } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class DataProcessingService {
  // Process array items as stream
  processItems(items: any[]): Observable<any> {
    return from(items).pipe(
      mergeMap(item => this.processItem(item)),
      toArray()
    );
  }
  
  private processItem(item: any): Observable<any> {
    // Process individual item
    return of(item).pipe(
      map(i => ({ ...i, processed: true }))
    );
  }
  
  // Convert IndexedDB promise to Observable
  getFromIndexedDB(key: string): Observable<any> {
    const request = indexedDB.open('myDatabase', 1);
    
    const promise = new Promise((resolve, reject) => {
      request.onsuccess = () => {
        const db = request.result;
        const transaction = db.transaction(['store'], 'readonly');
        const store = transaction.objectStore('store');
        const getRequest = store.get(key);
        
        getRequest.onsuccess = () => resolve(getRequest.result);
        getRequest.onerror = () => reject(getRequest.error);
      };
      request.onerror = () => reject(request.error);
    });
    
    return from(promise);
  }
}
```

## fromEvent Operator

`fromEvent` creates an Observable from DOM events or Node.js events:

```typescript
import { Component, ElementRef, OnInit, ViewChild } from '@angular/core';
import { fromEvent } from 'rxjs';
import { debounceTime, map } from 'rxjs/operators';

@Component({
  selector: 'app-event-demo',
  template: `
    <input #searchInput placeholder="Search...">
    <div>Clicks: {{ clickCount }}</div>
    <div>Scroll: {{ scrollY }}</div>
  `
})
export class EventDemoComponent implements OnInit {
  @ViewChild('searchInput', { static: true }) searchInput!: ElementRef;
  
  clickCount = 0;
  scrollY = 0;
  
  ngOnInit() {
    // Click events
    fromEvent(document, 'click').subscribe(() => {
      this.clickCount++;
    });
    
    // Scroll events with debounce
    fromEvent(window, 'scroll')
      .pipe(debounceTime(100))
      .subscribe(() => {
        this.scrollY = window.scrollY;
      });
    
    // Input events
    fromEvent(this.searchInput.nativeElement, 'input')
      .pipe(
        map((event: any) => event.target.value),
        debounceTime(300)
      )
      .subscribe(value => {
        console.log('Search for:', value);
      });
  }
}
```

### fromEvent with Options

```typescript
import { fromEvent } from 'rxjs';
import { tap } from 'rxjs/operators';

// With event listener options
const clicks$ = fromEvent(
  document,
  'click',
  { capture: true, passive: true }
);

// Custom event target
class CustomEmitter {
  private listeners: any[] = [];
  
  addEventListener(type: string, listener: any) {
    this.listeners.push(listener);
  }
  
  removeEventListener(type: string, listener: any) {
    const index = this.listeners.indexOf(listener);
    if (index > -1) {
      this.listeners.splice(index, 1);
    }
  }
  
  emit(data: any) {
    this.listeners.forEach(listener => listener(data));
  }
}

const emitter = new CustomEmitter();
const custom$ = fromEvent(emitter, 'data');

custom$.subscribe(data => console.log('Received:', data));

emitter.emit({ value: 123 });
```

### fromEvent for Keyboard Shortcuts

```typescript
import { Component, OnInit } from '@angular/core';
import { fromEvent } from 'rxjs';
import { filter, map } from 'rxjs/operators';

@Component({
  selector: 'app-shortcuts',
  template: `
    <div>Press Ctrl+S to save</div>
    <div>Press Ctrl+K to search</div>
    <div>Last action: {{ lastAction }}</div>
  `
})
export class ShortcutsComponent implements OnInit {
  lastAction = 'none';
  
  ngOnInit() {
    const keydown$ = fromEvent<KeyboardEvent>(document, 'keydown');
    
    // Ctrl+S to save
    keydown$
      .pipe(
        filter(event => event.ctrlKey && event.key === 's')
      )
      .subscribe(event => {
        event.preventDefault();
        this.lastAction = 'save';
        this.save();
      });
    
    // Ctrl+K to search
    keydown$
      .pipe(
        filter(event => event.ctrlKey && event.key === 'k')
      )
      .subscribe(event => {
        event.preventDefault();
        this.lastAction = 'search';
        this.openSearch();
      });
  }
  
  save() {
    console.log('Saving...');
  }
  
  openSearch() {
    console.log('Opening search...');
  }
}
```

## interval and timer

`interval` emits sequential numbers at specified intervals:

```typescript
import { interval } from 'rxjs';
import { take } from 'rxjs/operators';

// Emit every 1 second
const interval$ = interval(1000);

interval$.pipe(take(5)).subscribe({
  next: (value) => console.log('Interval:', value),
  complete: () => console.log('Complete')
});
// Emits: 0, 1, 2, 3, 4 (then completes)
```

### timer Operator

```typescript
import { timer } from 'rxjs';
import { take } from 'rxjs/operators';

// Emit once after 2 seconds
const onceTimer$ = timer(2000);
onceTimer$.subscribe(val => console.log('Timer:', val));
// Emits: 0 (after 2 seconds)

// Emit after 1 second, then every 500ms
const recurringTimer$ = timer(1000, 500);
recurringTimer$.pipe(take(5)).subscribe(val => console.log('Value:', val));
// Emits: 0 (after 1s), 1 (after 1.5s), 2 (after 2s), etc.
```

### Countdown Timer

```typescript
import { Component, OnInit } from '@angular/core';
import { timer, Observable } from 'rxjs';
import { map, takeWhile } from 'rxjs/operators';

@Component({
  selector: 'app-countdown',
  template: `
    <div class="countdown">
      <div>{{ timeLeft$ | async }}</div>
      <button (click)="start()" [disabled]="started">Start</button>
    </div>
  `
})
export class CountdownComponent {
  timeLeft$!: Observable<number>;
  started = false;
  
  start() {
    this.started = true;
    const duration = 10; // 10 seconds
    
    this.timeLeft$ = timer(0, 1000).pipe(
      map(n => duration - n),
      takeWhile(n => n >= 0)
    );
  }
}
```

### Polling with interval

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { interval, Subject } from 'rxjs';
import { switchMap, takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-polling',
  template: `
    <div>Data: {{ data | json }}</div>
    <div>Updated: {{ lastUpdate | date:'medium' }}</div>
  `
})
export class PollingComponent implements OnInit, OnDestroy {
  data: any = null;
  lastUpdate = new Date();
  
  private destroy$ = new Subject<void>();
  
  constructor(private http: HttpClient) {}
  
  ngOnInit() {
    // Poll every 5 seconds
    interval(5000)
      .pipe(
        switchMap(() => this.http.get('/api/data')),
        takeUntil(this.destroy$)
      )
      .subscribe(data => {
        this.data = data;
        this.lastUpdate = new Date();
      });
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## defer Operator

`defer` creates an Observable that's created fresh for each subscriber:

```typescript
import { defer, of } from 'rxjs';

// Without defer - timestamp created once
const eager$ = of(Date.now());

// With defer - timestamp created per subscription
const lazy$ = defer(() => of(Date.now()));

eager$.subscribe(val => console.log('Eager 1:', val));
setTimeout(() => {
  eager$.subscribe(val => console.log('Eager 2:', val));
}, 1000);
// Both get same timestamp

lazy$.subscribe(val => console.log('Lazy 1:', val));
setTimeout(() => {
  lazy$.subscribe(val => console.log('Lazy 2:', val));
}, 1000);
// Different timestamps
```

### defer for Conditional Observables

```typescript
import { defer, of, throwError } from 'rxjs';

function getDataObservable(useCache: boolean) {
  return defer(() => {
    if (useCache && cachedData) {
      console.log('Returning cached data');
      return of(cachedData);
    } else {
      console.log('Fetching fresh data');
      return http.get('/api/data');
    }
  });
}

// Decision made at subscription time
const data$ = getDataObservable(true);
data$.subscribe(data => console.log('Data:', data));
```

### defer for Retry Logic

```typescript
import { defer, throwError } from 'rxjs';
import { retry, tap } from 'rxjs/operators';

let attempt = 0;

const unreliable$ = defer(() => {
  attempt++;
  console.log('Attempt:', attempt);
  
  if (attempt < 3) {
    return throwError(() => new Error('Failed'));
  }
  
  return of('Success!');
});

unreliable$
  .pipe(retry(3))
  .subscribe({
    next: (value) => console.log('Result:', value),
    error: (err) => console.error('Error:', err)
  });
// Tries 3 times, succeeds on 3rd attempt
```

## range and generate

`range` emits a sequence of numbers:

```typescript
import { range } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// Emit numbers 1 to 10
range(1, 10).subscribe(val => console.log(val));

// Generate squares
range(1, 5)
  .pipe(map(n => n * n))
  .subscribe(val => console.log('Square:', val));
// Output: 1, 4, 9, 16, 25

// Filter even numbers
range(1, 10)
  .pipe(filter(n => n % 2 === 0))
  .subscribe(val => console.log('Even:', val));
// Output: 2, 4, 6, 8, 10
```

### generate Operator

```typescript
import { generate } from 'rxjs';

// Like a for loop as Observable
const generated$ = generate(
  1,              // Initial value
  x => x <= 5,    // Condition
  x => x + 1,     // Increment
  x => x * x      // Result selector
);

generated$.subscribe(val => console.log('Generated:', val));
// Output: 1, 4, 9, 16, 25

// Fibonacci sequence
const fibonacci$ = generate(
  [0, 1],
  ([a, b]) => a < 100,
  ([a, b]) => [b, a + b],
  ([a, b]) => a
);

fibonacci$.subscribe(val => console.log('Fib:', val));
// Output: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89
```

## Common Mistakes

### Mistake 1: Using of When You Need from

```typescript
// BAD: Emits array as single value
const bad$ = of([1, 2, 3, 4, 5]);
bad$.subscribe(val => console.log(val));
// Output: [1, 2, 3, 4, 5] (single emission)

// GOOD: Emits each item separately
const good$ = from([1, 2, 3, 4, 5]);
good$.subscribe(val => console.log(val));
// Output: 1, 2, 3, 4, 5 (separate emissions)
```

### Mistake 2: Not Unsubscribing from interval/timer

```typescript
// BAD: Memory leak
@Component({
  selector: 'app-leak',
  template: `<div>{{ count }}</div>`
})
export class LeakComponent implements OnInit {
  count = 0;
  
  ngOnInit() {
    interval(1000).subscribe(val => {
      this.count = val;
    });
    // Never unsubscribes!
  }
}

// GOOD: Proper cleanup
@Component({
  selector: 'app-no-leak',
  template: `<div>{{ count$ | async }}</div>`
})
export class NoLeakComponent {
  count$ = interval(1000);
  // Async pipe handles unsubscribe
}
```

### Mistake 3: Creating Observables in Template

```typescript
// BAD: Creates new Observable on every change detection
@Component({
  template: `
    <div>{{ of(value) | async }}</div>
  `
})
export class BadComponent {
  value = 42;
  
  of(val: any) {
    return of(val); // Creates new Observable constantly!
  }
}

// GOOD: Create Observable once
@Component({
  template: `
    <div>{{ value$ | async }}</div>
  `
})
export class GoodComponent {
  value$ = of(42); // Created once
}
```

## Best Practices

### 1. Use Appropriate Creation Operator

```typescript
// Synchronous values -> of
const sync$ = of(1, 2, 3);

// Array/Promise/Iterable -> from
const array$ = from([1, 2, 3]);
const promise$ = from(fetch('/api'));

// DOM events -> fromEvent
const clicks$ = fromEvent(button, 'click');

// Timed emissions -> interval/timer
const timed$ = interval(1000);

// Lazy/conditional -> defer
const lazy$ = defer(() => condition ? of(1) : of(2));
```

### 2. Always Manage Subscriptions

```typescript
// Use async pipe
@Component({
  template: `<div>{{ data$ | async }}</div>`
})
export class Component1 {
  data$ = interval(1000);
}

// Or takeUntilDestroyed
@Component({})
export class Component2 implements OnInit {
  private destroy$ = takeUntilDestroyed();
  
  ngOnInit() {
    interval(1000)
      .pipe(this.destroy$)
      .subscribe(val => console.log(val));
  }
}
```

### 3. Type Your Observables

```typescript
import { Observable, of, from } from 'rxjs';

interface User {
  id: number;
  name: string;
}

// Type-safe observables
const user$: Observable<User> = of({
  id: 1,
  name: 'John'
});

const users$: Observable<User> = from([
  { id: 1, name: 'John' },
  { id: 2, name: 'Jane' }
]);
```

## Interview Questions

### Q1: What's the difference between of and from?
**Answer:** `of` emits each argument as a separate value (of(1,2,3) emits 1, 2, 3). `from` converts an array, promise, or iterable to an Observable (from([1,2,3]) emits 1, 2, 3). Use `of` for known values, `from` for arrays/promises/iterables.

### Q2: When should you use defer?
**Answer:** Use `defer` when you want the Observable creation logic to run fresh for each subscriber. Common use cases: conditional observables, retry logic where state changes between attempts, or when you need different values per subscriber (like timestamps).

### Q3: What's the difference between interval and timer?
**Answer:** `interval(n)` starts emitting immediately at 0, then every n milliseconds. `timer(n)` waits n milliseconds before emitting 0. `timer(delay, period)` waits delay milliseconds, then emits periodically. Use timer for delayed start, interval for immediate periodic emissions.

### Q4: Why should fromEvent be cleaned up?
**Answer:** `fromEvent` creates an infinite Observable that listens to DOM events. Without unsubscribing, the event listener remains attached even after component destruction, causing memory leaks. Always use async pipe or unsubscribe in ngOnDestroy.

### Q5: Can you convert a Promise to an Observable?
**Answer:** Yes, use `from(promise)`. The Observable will emit the resolved value and complete, or error if the promise rejects. This is useful for using Promise-based APIs with RxJS operators like retry, timeout, or combining with other observables.

## Key Takeaways

1. Creation operators create Observables from various sources
2. `of` emits values in sequence, `from` converts iterables/promises
3. `fromEvent` converts DOM events to Observables
4. `interval` emits periodically, `timer` allows delayed start
5. `defer` creates fresh Observable per subscription
6. Always unsubscribe from infinite streams (interval, fromEvent)
7. Use async pipe for automatic subscription management
8. Choose the right creation operator for your data source
9. Type your Observables for type safety
10. `range` and `generate` create numeric sequences

## Resources

### Official Documentation
- [RxJS Creation Operators](https://rxjs.dev/guide/operators#creation-operators-1)
- [of API](https://rxjs.dev/api/index/function/of)
- [from API](https://rxjs.dev/api/index/function/from)
- [fromEvent API](https://rxjs.dev/api/index/function/fromEvent)

### Articles
- [Understanding Creation Operators](https://angular.io/guide/observables)
- [RxJS Operators Guide](https://rxjs.dev/guide/operators)

### Tools
- RxJS Marbles Diagram
- RxViz for visualization
