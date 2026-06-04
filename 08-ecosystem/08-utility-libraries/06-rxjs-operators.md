# RxJS Operators: Reactive Programming Guide

## Overview

RxJS (Reactive Extensions for JavaScript) is a library for reactive programming using Observables, making it easier to compose asynchronous or callback-based code. Operators are the backbone of RxJS, providing powerful tools to transform, filter, combine, and manage data streams.

**Key Features:**
- 100+ operators for stream manipulation
- Declarative, composable data flow
- Backpressure handling
- Memory management through unsubscription
- Error handling operators
- Time-based operations
- Hot and cold Observables
- Multicasting capabilities
- Scheduler control for async operations
- TypeScript support with full type safety

RxJS is the foundation for Angular's reactive patterns and is widely used for complex async operations, real-time data, and event handling in web applications.

## Installation and Setup

### Installation

```bash
# npm
npm install rxjs

# yarn
yarn add rxjs

# pnpm
pnpm add rxjs
```

### Basic Imports

```javascript
// Import Observable and operators
import { Observable, Subject, BehaviorSubject } from 'rxjs';
import { map, filter, tap, catchError, retry } from 'rxjs/operators';

// Creation operators
import { of, from, interval, fromEvent } from 'rxjs';

// All operators in one (not recommended for production)
import * as rxjs from 'rxjs';
import * as operators from 'rxjs/operators';
```

### Creating Observables

```javascript
import { Observable, of, from, interval, fromEvent } from 'rxjs';

// From values
of(1, 2, 3, 4, 5).subscribe(console.log); // 1, 2, 3, 4, 5

// From array
from([1, 2, 3, 4, 5]).subscribe(console.log);

// From promise
from(fetch('/api/data').then(r => r.json()))
  .subscribe(data => console.log(data));

// Timer
interval(1000).subscribe(count => console.log(count)); // 0, 1, 2, 3...

// From events
const clicks$ = fromEvent(document, 'click');
clicks$.subscribe(event => console.log('Clicked!', event));

// Custom Observable
const custom$ = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  setTimeout(() => {
    subscriber.next(3);
    subscriber.complete();
  }, 1000);
});
```

## Core Concepts

### 1. Observables and Observers

```javascript
import { Observable } from 'rxjs';

// Create Observable
const observable$ = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
  
  // Cleanup function
  return () => {
    console.log('Unsubscribed');
  };
});

// Subscribe with Observer
const subscription = observable$.subscribe({
  next: value => console.log('Next:', value),
  error: err => console.error('Error:', err),
  complete: () => console.log('Complete!')
});

// Unsubscribe
setTimeout(() => {
  subscription.unsubscribe();
}, 500);
```

### 2. Pipeable Operators

```javascript
import { of } from 'rxjs';
import { map, filter, tap } from 'rxjs/operators';

// Chain operators with pipe
of(1, 2, 3, 4, 5)
  .pipe(
    tap(x => console.log('Before:', x)),
    filter(x => x % 2 === 0),
    map(x => x * 10),
    tap(x => console.log('After:', x))
  )
  .subscribe(console.log);

// Output:
// Before: 1
// Before: 2
// After: 20
// 20
// Before: 3
// Before: 4
// After: 40
// 40
// Before: 5
```

### 3. Hot vs Cold Observables

```javascript
import { Observable, Subject } from 'rxjs';

// Cold Observable (creates new execution per subscriber)
const cold$ = new Observable(subscriber => {
  console.log('Cold Observable execution');
  subscriber.next(Math.random());
});

cold$.subscribe(x => console.log('Sub 1:', x)); // New execution
cold$.subscribe(x => console.log('Sub 2:', x)); // New execution
// Different random numbers

// Hot Observable (shares single execution)
const subject = new Subject();
subject.subscribe(x => console.log('Sub 1:', x));
subject.subscribe(x => console.log('Sub 2:', x));
subject.next(Math.random()); // Same value for both
```

## Creation Operators

### Observable Creation

```javascript
import { 
  of, from, fromEvent, interval, timer,
  range, generate, ajax, defer
} from 'rxjs';

// of - emit values in sequence
of(1, 2, 3, 4, 5).subscribe(console.log);

// from - convert array/promise/iterable to Observable
from([1, 2, 3]).subscribe(console.log);
from(Promise.resolve('Hello')).subscribe(console.log);

// fromEvent - DOM events
const clicks$ = fromEvent(button, 'click');
clicks$.subscribe(event => console.log('Clicked'));

// interval - emit every N milliseconds
interval(1000).subscribe(count => console.log(count)); // 0, 1, 2...

// timer - emit after delay, then at intervals
timer(2000, 1000).subscribe(count => console.log(count));
// Wait 2s, then emit 0, 1, 2... every 1s

// range - emit range of numbers
range(1, 5).subscribe(console.log); // 1, 2, 3, 4, 5

// generate - conditional generation
generate(
  0,                    // Initial value
  x => x < 10,         // Condition
  x => x + 2           // Increment
).subscribe(console.log); // 0, 2, 4, 6, 8

// defer - Create Observable lazily
const deferred$ = defer(() => {
  return Math.random() > 0.5 ? of('A') : of('B');
});
```

## Transformation Operators

### map, pluck, mapTo

```javascript
import { of, fromEvent } from 'rxjs';
import { map, pluck, mapTo } from 'rxjs/operators';

// map - transform each value
of(1, 2, 3, 4, 5)
  .pipe(map(x => x * 10))
  .subscribe(console.log); // 10, 20, 30, 40, 50

// pluck - extract nested property (deprecated, use map)
const users$ = of(
  { name: 'Alice', age: 30 },
  { name: 'Bob', age: 25 }
);

users$
  .pipe(map(user => user.name))
  .subscribe(console.log); // 'Alice', 'Bob'

// mapTo - replace with constant value
fromEvent(button, 'click')
  .pipe(mapTo(1))
  .subscribe(console.log); // 1, 1, 1... for each click
```

### mergeMap, switchMap, concatMap, exhaustMap

```javascript
import { of, fromEvent, interval } from 'rxjs';
import { mergeMap, switchMap, concatMap, exhaustMap, take } from 'rxjs/operators';

const clicks$ = fromEvent(button, 'click');

// mergeMap - run all inner Observables concurrently
clicks$
  .pipe(
    mergeMap(() => interval(1000).pipe(take(3)))
  )
  .subscribe(console.log);
// Multiple intervals run simultaneously

// switchMap - cancel previous, switch to new
clicks$
  .pipe(
    switchMap(() => interval(1000).pipe(take(3)))
  )
  .subscribe(console.log);
// Each click cancels previous interval

// concatMap - queue inner Observables
clicks$
  .pipe(
    concatMap(() => interval(1000).pipe(take(3)))
  )
  .subscribe(console.log);
// Wait for each interval to complete before next

// exhaustMap - ignore new until current completes
clicks$
  .pipe(
    exhaustMap(() => interval(1000).pipe(take(3)))
  )
  .subscribe(console.log);
// Ignore clicks while interval is running

// Practical example: Search API
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

searchInput$
  .pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(query => 
      fetch(`/api/search?q=${query}`).then(r => r.json())
    )
  )
  .subscribe(results => displayResults(results));
```

### scan, reduce

```javascript
import { of, fromEvent } from 'rxjs';
import { scan, reduce, map } from 'rxjs/operators';

// scan - accumulate over time (like Array.reduce but emits intermediate)
of(1, 2, 3, 4, 5)
  .pipe(
    scan((acc, curr) => acc + curr, 0)
  )
  .subscribe(console.log);
// 1, 3, 6, 10, 15 (running total)

// reduce - emit only final accumulated value
of(1, 2, 3, 4, 5)
  .pipe(
    reduce((acc, curr) => acc + curr, 0)
  )
  .subscribe(console.log);
// 15 (final total only)

// Practical: Click counter
fromEvent(button, 'click')
  .pipe(
    scan(count => count + 1, 0)
  )
  .subscribe(count => {
    counter.textContent = count;
  });
```

## Filtering Operators

### filter, take, skip, distinct

```javascript
import { of, interval } from 'rxjs';
import { 
  filter, take, takeWhile, takeUntil, 
  skip, skipWhile, skipUntil,
  distinct, distinctUntilChanged
} from 'rxjs/operators';

// filter - emit values that pass condition
of(1, 2, 3, 4, 5)
  .pipe(filter(x => x % 2 === 0))
  .subscribe(console.log); // 2, 4

// take - take first N values
interval(1000)
  .pipe(take(5))
  .subscribe(console.log); // 0, 1, 2, 3, 4, then complete

// takeWhile - take while condition is true
of(1, 2, 3, 4, 5, 3, 2, 1)
  .pipe(takeWhile(x => x < 4))
  .subscribe(console.log); // 1, 2, 3

// takeUntil - take until notifier emits
const stop$ = fromEvent(stopButton, 'click');
interval(1000)
  .pipe(takeUntil(stop$))
  .subscribe(console.log); // Stop when button clicked

// skip - skip first N values
of(1, 2, 3, 4, 5)
  .pipe(skip(2))
  .subscribe(console.log); // 3, 4, 5

// skipWhile - skip while condition is true
of(1, 2, 3, 4, 5, 3, 2, 1)
  .pipe(skipWhile(x => x < 4))
  .subscribe(console.log); // 4, 5, 3, 2, 1

// distinct - emit only unique values
of(1, 2, 2, 3, 3, 3, 4, 5, 5)
  .pipe(distinct())
  .subscribe(console.log); // 1, 2, 3, 4, 5

// distinctUntilChanged - emit only when value changes
of(1, 1, 2, 2, 3, 3, 2, 1)
  .pipe(distinctUntilChanged())
  .subscribe(console.log); // 1, 2, 3, 2, 1
```

### debounceTime, throttleTime, auditTime, sampleTime

```javascript
import { fromEvent, interval } from 'rxjs';
import { debounceTime, throttleTime, auditTime, sampleTime } from 'rxjs/operators';

const input$ = fromEvent(inputElement, 'input');

// debounceTime - emit after silence period
input$
  .pipe(debounceTime(300))
  .subscribe(event => {
    // Only fires 300ms after user stops typing
    console.log(event.target.value);
  });

// throttleTime - emit first, then ignore for duration
fromEvent(button, 'click')
  .pipe(throttleTime(1000))
  .subscribe(() => {
    // Max one click per second
    console.log('Clicked');
  });

// auditTime - emit last value after duration
input$
  .pipe(auditTime(300))
  .subscribe(event => {
    // Emit last value every 300ms
    console.log(event.target.value);
  });

// sampleTime - emit most recent value at intervals
interval(100)
  .pipe(sampleTime(1000))
  .subscribe(console.log);
// Emit latest value every 1 second
```

## Combination Operators

### merge, concat, race, zip, combineLatest

```javascript
import { merge, concat, race, zip, combineLatest, interval, of } from 'rxjs';
import { map, take } from 'rxjs/operators';

// merge - emit from all Observables concurrently
const first$ = interval(1000).pipe(map(x => `First: ${x}`), take(3));
const second$ = interval(1500).pipe(map(x => `Second: ${x}`), take(3));

merge(first$, second$).subscribe(console.log);
// First: 0
// First: 1
// Second: 0
// First: 2
// Second: 1
// Second: 2

// concat - emit from Observables sequentially
concat(first$, second$).subscribe(console.log);
// Wait for first$ to complete, then emit from second$

// race - emit from Observable that emits first
race(first$, second$).subscribe(console.log);
// Only emit from first$ (it emits first)

// zip - combine corresponding values
const ages$ = of(25, 30, 35);
const names$ = of('Alice', 'Bob', 'Charlie');

zip(names$, ages$)
  .pipe(map(([name, age]) => ({ name, age })))
  .subscribe(console.log);
// { name: 'Alice', age: 25 }
// { name: 'Bob', age: 30 }
// { name: 'Charlie', age: 35 }

// combineLatest - combine latest values from all
const temp$ = of(20, 22, 21);
const humidity$ = of(60, 65);

combineLatest([temp$, humidity$])
  .pipe(map(([temp, humidity]) => ({ temp, humidity })))
  .subscribe(console.log);
// { temp: 20, humidity: 60 }
// { temp: 22, humidity: 60 }
// { temp: 22, humidity: 65 }
// { temp: 21, humidity: 65 }
```

### withLatestFrom, startWith, endWith

```javascript
import { fromEvent, interval } from 'rxjs';
import { withLatestFrom, startWith, endWith, map } from 'rxjs/operators';

// withLatestFrom - include latest value from other Observable
const clicks$ = fromEvent(button, 'click');
const timer$ = interval(1000);

clicks$
  .pipe(
    withLatestFrom(timer$),
    map(([click, time]) => ({ click, time }))
  )
  .subscribe(console.log);
// Each click includes current timer value

// startWith - emit initial value(s)
of(4, 5, 6)
  .pipe(startWith(1, 2, 3))
  .subscribe(console.log);
// 1, 2, 3, 4, 5, 6

// endWith - emit final value(s)
of(1, 2, 3)
  .pipe(endWith(4, 5, 6))
  .subscribe(console.log);
// 1, 2, 3, 4, 5, 6
```

## Error Handling Operators

### catchError, retry, retryWhen

```javascript
import { of, throwError, timer } from 'rxjs';
import { catchError, retry, retryWhen, mergeMap, tap } from 'rxjs/operators';

// catchError - handle errors
of(1, 2, 3)
  .pipe(
    map(x => {
      if (x === 2) throw new Error('Error!');
      return x;
    }),
    catchError(err => {
      console.error('Caught:', err.message);
      return of('Recovered');
    })
  )
  .subscribe(console.log);
// 1, 'Recovered', 3

// retry - retry on error
const unstable$ = new Observable(subscriber => {
  const random = Math.random();
  if (random < 0.7) {
    subscriber.error('Failed');
  } else {
    subscriber.next('Success');
    subscriber.complete();
  }
});

unstable$
  .pipe(retry(3)) // Retry up to 3 times
  .subscribe({
    next: console.log,
    error: err => console.error('Still failed after retries')
  });

// retryWhen - retry with strategy
fetch('/api/data')
  .pipe(
    retryWhen(errors => 
      errors.pipe(
        tap(err => console.log('Retrying...', err)),
        mergeMap((err, i) => {
          // Exponential backoff
          if (i >= 3) {
            return throwError('Max retries reached');
          }
          return timer(Math.pow(2, i) * 1000);
        })
      )
    ),
    catchError(err => {
      console.error('Final error:', err);
      return of({ error: true });
    })
  )
  .subscribe(data => console.log(data));
```

## Utility Operators

### tap, delay, timeout, finalize

```javascript
import { of, timer } from 'rxjs';
import { tap, delay, delayWhen, timeout, finalize } from 'rxjs/operators';

// tap - perform side effect
of(1, 2, 3)
  .pipe(
    tap(x => console.log('Before:', x)),
    map(x => x * 10),
    tap(x => console.log('After:', x))
  )
  .subscribe(console.log);

// delay - delay emission by time
of('Hello')
  .pipe(delay(2000))
  .subscribe(console.log); // After 2 seconds

// delayWhen - delay each emission by Observable
of(1, 2, 3)
  .pipe(
    delayWhen(x => timer(x * 1000))
  )
  .subscribe(console.log);
// 1 after 1s, 2 after 2s, 3 after 3s

// timeout - error if no emission within time
of('Hello')
  .pipe(
    delay(3000),
    timeout(2000)
  )
  .subscribe({
    next: console.log,
    error: err => console.error('Timeout!')
  });

// finalize - cleanup logic
of(1, 2, 3)
  .pipe(
    tap(console.log),
    finalize(() => console.log('Cleanup'))
  )
  .subscribe();
// 1, 2, 3, 'Cleanup'
```

## Multicasting Operators

### share, shareReplay, publish, multicast

```javascript
import { interval, Subject } from 'rxjs';
import { 
  share, shareReplay, publish, multicast, refCount,
  tap, take 
} from 'rxjs/operators';

// share - share execution among subscribers
const shared$ = interval(1000).pipe(
  tap(() => console.log('Execution')),
  take(5),
  share()
);

shared$.subscribe(x => console.log('Sub 1:', x));
setTimeout(() => {
  shared$.subscribe(x => console.log('Sub 2:', x));
}, 2500);
// Only one execution, both subscribers get values

// shareReplay - share and replay last N values
const replayed$ = interval(1000).pipe(
  tap(() => console.log('Execution')),
  take(5),
  shareReplay(2) // Replay last 2 values
);

replayed$.subscribe(x => console.log('Sub 1:', x));
setTimeout(() => {
  replayed$.subscribe(x => console.log('Sub 2:', x));
  // Gets last 2 values immediately
}, 3500);

// publish - manual control with ConnectableObservable
const published$ = interval(1000).pipe(
  tap(() => console.log('Execution')),
  take(5),
  publish()
);

published$.subscribe(x => console.log('Sub 1:', x));
published$.subscribe(x => console.log('Sub 2:', x));
published$.connect(); // Start execution manually
```

## Subjects

### Subject, BehaviorSubject, ReplaySubject, AsyncSubject

```javascript
import { Subject, BehaviorSubject, ReplaySubject, AsyncSubject } from 'rxjs';

// Subject - multicast to multiple observers
const subject = new Subject();

subject.subscribe(x => console.log('Sub 1:', x));
subject.subscribe(x => console.log('Sub 2:', x));

subject.next(1); // Both receive 1
subject.next(2); // Both receive 2

// BehaviorSubject - holds current value
const behavior = new BehaviorSubject(0); // Initial value

behavior.subscribe(x => console.log('Sub 1:', x)); // Immediately gets 0
behavior.next(1);
behavior.next(2);
behavior.subscribe(x => console.log('Sub 2:', x)); // Gets current value: 2
behavior.next(3); // Both receive 3

console.log(behavior.value); // Access current value: 3

// ReplaySubject - replays last N values
const replay = new ReplaySubject(2); // Buffer size: 2

replay.next(1);
replay.next(2);
replay.next(3);

replay.subscribe(x => console.log('Sub:', x));
// Immediately gets: 2, 3 (last 2 values)

replay.next(4); // Sub receives 4

// AsyncSubject - emits only last value on complete
const async = new AsyncSubject();

async.subscribe(x => console.log('Sub 1:', x));

async.next(1);
async.next(2);
async.next(3);
async.complete(); // Only now emits: 3

async.subscribe(x => console.log('Sub 2:', x)); // Also gets 3
```

## Practical Examples

### Form Validation

```javascript
import { fromEvent, combineLatest } from 'rxjs';
import { map, debounceTime, distinctUntilChanged } from 'rxjs/operators';

const emailInput$ = fromEvent(emailInput, 'input').pipe(
  map(e => e.target.value),
  debounceTime(300),
  distinctUntilChanged()
);

const passwordInput$ = fromEvent(passwordInput, 'input').pipe(
  map(e => e.target.value),
  debounceTime(300),
  distinctUntilChanged()
);

// Validate email
const isEmailValid$ = emailInput$.pipe(
  map(email => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email))
);

// Validate password
const isPasswordValid$ = passwordInput$.pipe(
  map(password => password.length >= 8)
);

// Combine validations
const isFormValid$ = combineLatest([isEmailValid$, isPasswordValid$]).pipe(
  map(([emailValid, passwordValid]) => emailValid && passwordValid)
);

isFormValid$.subscribe(valid => {
  submitButton.disabled = !valid;
});
```

### Real-Time Search

```javascript
import { fromEvent, of } from 'rxjs';
import { 
  debounceTime, distinctUntilChanged, switchMap,
  map, catchError, tap
} from 'rxjs/operators';

const searchInput$ = fromEvent(searchInput, 'input').pipe(
  map(e => e.target.value),
  debounceTime(300),
  distinctUntilChanged(),
  tap(() => showLoading()),
  switchMap(query => {
    if (query.length < 2) {
      return of([]);
    }
    
    return fetch(`/api/search?q=${encodeURIComponent(query)}`)
      .then(r => r.json())
      .pipe(
        catchError(err => {
          console.error('Search error:', err);
          return of([]);
        })
      );
  }),
  tap(() => hideLoading())
);

searchInput$.subscribe(results => {
  displayResults(results);
});
```

### Drag and Drop

```javascript
import { fromEvent, merge } from 'rxjs';
import { 
  map, switchMap, takeUntil, tap
} from 'rxjs/operators';

const element = document.getElementById('draggable');

const mouseDown$ = fromEvent(element, 'mousedown');
const mouseMove$ = fromEvent(document, 'mousemove');
const mouseUp$ = fromEvent(document, 'mouseup');

const drag$ = mouseDown$.pipe(
  tap(e => e.preventDefault()),
  map(down => ({
    startX: down.clientX - element.offsetLeft,
    startY: down.clientY - element.offsetTop
  })),
  switchMap(start => 
    mouseMove$.pipe(
      map(move => ({
        x: move.clientX - start.startX,
        y: move.clientY - start.startY
      })),
      takeUntil(mouseUp$)
    )
  )
);

drag$.subscribe(pos => {
  element.style.left = pos.x + 'px';
  element.style.top = pos.y + 'px';
});
```

### WebSocket with Reconnection

```javascript
import { webSocket } from 'rxjs/webSocket';
import { retry, retryWhen, delay, tap } from 'rxjs/operators';

const socket$ = webSocket({
  url: 'ws://localhost:8080',
  openObserver: {
    next: () => console.log('Connection established')
  },
  closeObserver: {
    next: () => console.log('Connection closed')
  }
}).pipe(
  retryWhen(errors =>
    errors.pipe(
      tap(err => console.error('WebSocket error:', err)),
      delay(5000), // Wait 5s before reconnecting
      tap(() => console.log('Reconnecting...'))
    )
  )
);

socket$.subscribe(
  message => console.log('Received:', message),
  error => console.error('Error:', error),
  () => console.log('Complete')
);

// Send message
socket$.next({ type: 'message', data: 'Hello' });
```

## Common Mistakes

### 1. Not Unsubscribing

```javascript
// Wrong: Memory leak
interval(1000).subscribe(console.log);

// Correct: Save and unsubscribe
const subscription = interval(1000).subscribe(console.log);

// Clean up
ngOnDestroy() {
  subscription.unsubscribe();
}

// Or use takeUntil
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

private destroy$ = new Subject();

ngOnInit() {
  interval(1000)
    .pipe(takeUntil(this.destroy$))
    .subscribe(console.log);
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### 2. Using Wrong Flattening Operator

```javascript
// Wrong: Using mergeMap for HTTP requests
searchInput$
  .pipe(
    mergeMap(query => this.http.get(`/api/search?q=${query}`))
  )
  .subscribe(results => {});
// All requests run concurrently, old results may arrive late!

// Correct: Use switchMap
searchInput$
  .pipe(
    switchMap(query => this.http.get(`/api/search?q=${query}`))
  )
  .subscribe(results => {});
// Cancels previous request on new input
```

### 3. Subscribing Inside Subscribe

```javascript
// Wrong: Nested subscriptions (callback hell)
observable1$.subscribe(data1 => {
  observable2$.subscribe(data2 => {
    observable3$.subscribe(data3 => {
      // Use data1, data2, data3
    });
  });
});

// Correct: Use operators
observable1$
  .pipe(
    switchMap(data1 => 
      observable2$.pipe(
        map(data2 => ({ data1, data2 }))
      )
    ),
    switchMap(({ data1, data2 }) =>
      observable3$.pipe(
        map(data3 => ({ data1, data2, data3 }))
      )
    )
  )
  .subscribe(({ data1, data2, data3 }) => {
    // Use all data
  });
```

### 4. Not Handling Errors

```javascript
// Wrong: No error handling
observable$
  .subscribe(data => console.log(data));
// Unhandled error breaks stream!

// Correct: Handle errors
observable$
  .pipe(
    catchError(err => {
      console.error('Error:', err);
      return of(defaultValue);
    })
  )
  .subscribe(data => console.log(data));
```

## Best Practices

### 1. Use takeUntil for Component Cleanup

```typescript
import { Component, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({/* ... */})
export class MyComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    this.dataService.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => {});
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. Name Observables with $ Suffix

```javascript
// Convention: Observable names end with $
const clicks$ = fromEvent(button, 'click');
const data$ = http.get('/api/data');
const timer$ = interval(1000);

// Makes it clear what's an Observable
```

### 3. Use shareReplay for Expensive Operations

```javascript
import { shareReplay } from 'rxjs/operators';

// Expensive API call
const expensiveData$ = http.get('/api/expensive')
  .pipe(
    shareReplay(1) // Cache and share result
  );

// Multiple subscribers use same result
expensiveData$.subscribe(data => {});
expensiveData$.subscribe(data => {}); // No second request!
```

### 4. Prefer Declarative over Imperative

```javascript
// Imperative (bad)
let count = 0;
clicks$.subscribe(() => {
  count++;
  console.log(count);
});

// Declarative (good)
const count$ = clicks$.pipe(
  scan(count => count + 1, 0)
);
count$.subscribe(console.log);
```

## Interview Questions

### 1. What's the difference between mergeMap, switchMap, concatMap, and exhaustMap?

**Answer:** All flatten higher-order Observables but differ in concurrency handling. mergeMap runs all inner Observables concurrently (use for independent operations). switchMap cancels previous inner Observable when new one arrives (use for search/typeahead). concatMap queues inner Observables, waiting for each to complete (use for ordered operations). exhaustMap ignores new values while current inner Observable runs (use for preventing duplicate actions like button clicks).

### 2. What's the difference between Subject, BehaviorSubject, and ReplaySubject?

**Answer:** Subject is a basic multicast Observable with no initial value or memory. BehaviorSubject holds current value (requires initial value), new subscribers immediately receive it. ReplaySubject buffers N last values (or all with no limit), new subscribers receive buffered values. AsyncSubject emits only last value on complete. Use Subject for event emitters, BehaviorSubject for state, ReplaySubject for caching, AsyncSubject for single async results.

### 3. How do you handle memory leaks in RxJS?

**Answer:** Always unsubscribe from subscriptions in component cleanup. Use takeUntil with destroy$ Subject pattern for automatic cleanup. Use async pipe in Angular (auto-unsubscribes). Avoid subscribing in subscribe (nested subscriptions). Let RxJS operators handle subscription management. For finite streams (take, first), they complete automatically. Memory leaks occur when infinite Observables (interval, fromEvent) aren't unsubscribed.

### 4. When should you use cold vs hot Observables?

**Answer:** Cold Observables create new execution for each subscriber (HTTP requests, timers created with interval). Use for independent data streams where each subscriber needs separate execution. Hot Observables share execution among subscribers (DOM events, WebSockets, Subjects). Use for shared events or state where all subscribers receive same values. Convert cold to hot with share/shareReplay when multiple subscribers should share execution.

### 5. How do error handling operators work in RxJS?

**Answer:** catchError catches errors and returns fallback Observable (recovery). retry automatically resubscribes N times on error (simple retry). retryWhen provides custom retry strategy (exponential backoff). Errors unhandled in subscribe break the stream permanently. Handle errors at appropriate level: in operators for recoverable errors, in subscribe for final handling. Error in inner Observable (mergeMap) doesn't break outer stream if caught.

### 6. What are marble diagrams and how do you read them?

**Answer:** Marble diagrams visualize Observable behavior over time. Timeline flows left to right. Circles (marbles) represent emitted values. Vertical line (|) represents completion. X represents error. Operators shown in the middle transform input to output. For example: `---1---2---3-|` shows values 1, 2, 3 emitted over time then completion. Useful for understanding operator behavior, especially combination and time-based operators.

### 7. How do you implement undo/redo with RxJS?

**Answer:** Use scan to accumulate history of states. Store past and future arrays. On action, push current state to past, clear future. On undo, pop from past, push to future. On redo, pop from future, push to past. Use Subject to trigger undo/redo actions. Combine with distinctUntilChanged to avoid duplicate states. For complex apps, consider recording actions/patches instead of full state copies for efficiency.

## Key Takeaways

1. **Operators enable** declarative, composable data transformation pipelines
2. **Flattening operators** (mergeMap, switchMap, concatMap, exhaustMap) crucial for async operations
3. **Memory management** requires proper unsubscription to prevent leaks
4. **Error handling** must be done in operators (catchError, retry) to keep streams alive
5. **Subjects** bridge imperative and reactive code, enabling multicasting
6. **Hot vs cold** distinction important for understanding execution behavior
7. **Combination operators** (combineLatest, zip, merge) enable complex data flows
8. **Time operators** (debounceTime, throttleTime) essential for UI events
9. **shareReplay** caches expensive operations for multiple subscribers
10. **takeUntil pattern** is best practice for Angular component cleanup

## Resources

- **Official Documentation**: https://rxjs.dev/
- **Learn RxJS**: https://www.learnrxjs.io/
- **RxJS Marbles**: https://rxmarbles.com/ (interactive marble diagrams)
- **RxJS Operator Decision Tree**: Choose right operator
- **GitHub Repository**: https://github.com/ReactiveX/rxjs
- **RxJS in Angular**: Angular-specific patterns
- **Reactive Programming**: Understanding reactive paradigm
- **Ben Lesh Talks**: Core team member presentations
- **RxJS Visualizer**: Debug Observable streams visually
- **Stack Overflow**: Common RxJS questions and answers
