# Marble Testing for RxJS Observables in Angular

## Table of Contents
- [Introduction](#introduction)
- [Understanding Marble Diagrams](#understanding-marble-diagrams)
- [TestScheduler Fundamentals](#testscheduler-fundamentals)
- [Basic Marble Testing](#basic-marble-testing)
- [Testing Hot vs Cold Observables](#testing-hot-vs-cold-observables)
- [Testing Time-Based Operations](#testing-time-based-operations)
- [Testing Error Scenarios](#testing-error-scenarios)
- [Testing Angular Services with Marble Testing](#testing-angular-services-with-marble-testing)
- [Advanced Marble Testing Patterns](#advanced-marble-testing-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Marble testing is a powerful technique for testing RxJS observables using visual marble diagrams. It allows you to test complex asynchronous operations synchronously by virtualizing time, making tests faster, more reliable, and easier to understand. This is especially valuable in Angular applications where RxJS is used extensively for state management, HTTP requests, and reactive forms.

Marble testing uses ASCII diagrams to represent observable streams over time, making it intuitive to visualize and test operator behavior, timing, and error handling.

## Understanding Marble Diagrams

### Marble Diagram Syntax

```typescript
// Basic marble syntax
'-'     // Time frame (10ms by default)
'|'     // Completion
'#'     // Error
'a'     // Emitted value
'(ab)'  // Synchronous grouped emissions
'^'     // Subscription point (hot observables)
'!'     // Unsubscription point

// Examples:
'----a---b---c---|'         // Emit a, b, c then complete
'----a---b---c---#'         // Emit a, b, c then error
'----a---b---(cd)|'         // Emit a, b, then c and d synchronously, then complete
'--a--b--c--d--e--f--|'     // Regular emissions with completion
```

### Time Progression

```typescript
// Each dash represents a virtual time frame (default 10ms)
'---a---|'  // Emit 'a' at 30ms, complete at 70ms
'a-b-c|'    // Emit 'a' at 0ms, 'b' at 20ms, 'c' at 40ms, complete at 50ms
'(abc)|'    // Emit a, b, c synchronously at 0ms, complete at 0ms
```

## TestScheduler Fundamentals

### Basic TestScheduler Setup

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('Marble Testing Basics', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      // Custom assertion
      expect(actual).toEqual(expected);
    });
  });

  it('should create a simple observable', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c--|');
      const expected = '--a--b--c--|';

      expectObservable(source$).toBe(expected);
    });
  });
});
```

### Helper Functions

```typescript
testScheduler.run(helpers => {
  const {
    cold,           // Create cold observable
    hot,            // Create hot observable
    expectObservable, // Assert observable behavior
    expectSubscriptions, // Assert subscription timing
    flush,          // Manually trigger virtual time
    time,           // Get time in frames
  } = helpers;

  // Use helpers here
});
```

## Basic Marble Testing

### Testing Simple Operators

```typescript
import { map } from 'rxjs/operators';

describe('Basic Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should map values', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c--|', { a: 1, b: 2, c: 3 });
      const expected =     '--a--b--c--|';
      const values = { a: 2, b: 4, c: 6 };

      const result$ = source$.pipe(
        map(x => x * 2)
      );

      expectObservable(result$).toBe(expected, values);
    });
  });

  it('should filter values', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c--d--|', { a: 1, b: 2, c: 3, d: 4 });
      const expected =     '-----b-----d--|';
      const values = { b: 2, d: 4 };

      const result$ = source$.pipe(
        filter(x => x % 2 === 0)
      );

      expectObservable(result$).toBe(expected, values);
    });
  });
});
```

### Testing with Values

```typescript
it('should transform objects', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--|', {
      a: { id: 1, name: 'John' },
      b: { id: 2, name: 'Jane' }
    });

    const expected = '--a--b--|';
    const values = {
      a: 'JOHN',
      b: 'JANE'
    };

    const result$ = source$.pipe(
      map(user => user.name.toUpperCase())
    );

    expectObservable(result$).toBe(expected, values);
  });
});
```

### Testing Completion and Errors

```typescript
it('should handle completion', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--|');
    const expected =     '--a--b--|';

    expectObservable(source$).toBe(expected);
  });
});

it('should handle errors', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--#');
    const expected =     '--a--b--#';

    expectObservable(source$).toBe(expected);
  });
});
```

## Testing Hot vs Cold Observables

### Cold Observables

```typescript
it('should test cold observable - each subscription starts from beginning', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--c--|');

    // First subscription
    const expected1 = '--a--b--c--|';
    expectObservable(source$).toBe(expected1);

    // Second subscription (independent)
    const expected2 = '--a--b--c--|';
    expectObservable(source$).toBe(expected2);
  });
});
```

### Hot Observables

```typescript
it('should test hot observable - subscription point matters', () => {
  testScheduler.run(({ hot, expectObservable }) => {
    // ^ indicates subscription point
    const source$ = hot('--a--b--^-c--d--|');
    const expected =            '--c--d--|';

    expectObservable(source$).toBe(expected);
  });
});

it('should test multiple subscriptions to hot observable', () => {
  testScheduler.run(({ hot, expectObservable }) => {
    const source$ = hot('--a--b--c--d--e--|');

    // First subscription from beginning
    const sub1 =        '^----------!';
    const expected1 =   '--a--b--c--d';

    // Second subscription starts later
    const sub2 =        '-----^----------!';
    const expected2 =   '-----b--c--d--e-';

    expectObservable(source$, sub1).toBe(expected1);
    expectObservable(source$, sub2).toBe(expected2);
  });
});
```

### Testing Subject Behavior

```typescript
import { Subject } from 'rxjs';

it('should test Subject as hot observable', () => {
  testScheduler.run(({ hot, expectObservable }) => {
    const subject = new Subject<string>();

    // Simulate emissions
    const source$ = hot('--a--b--c--|', { a: 'A', b: 'B', c: 'C' });
    source$.subscribe(val => subject.next(val));

    const expected = '--a--b--c--|';
    expectObservable(subject).toBe(expected, { a: 'A', b: 'B', c: 'C' });
  });
});
```

## Testing Time-Based Operations

### Testing debounceTime

```typescript
import { debounceTime } from 'rxjs/operators';

it('should debounce emissions', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    // User typing simulation
    const source$ = cold('a-b-c----d------|');
    const expected =     '----c-------d---|';

    const result$ = source$.pipe(
      debounceTime(30) // 3 frames
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing throttleTime

```typescript
import { throttleTime } from 'rxjs/operators';

it('should throttle emissions', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('a-b-c-d-e-f-g-h-|');
    const expected =     'a---d---g------|';

    const result$ = source$.pipe(
      throttleTime(30) // 3 frames
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing delay

```typescript
import { delay } from 'rxjs/operators';

it('should delay emissions', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--c--|');
    const expected =     '----a--b--c--|';

    const result$ = source$.pipe(
      delay(20) // 2 frames
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing interval and timer

```typescript
import { interval, take } from 'rxjs';

it('should test interval', () => {
  testScheduler.run(({ expectObservable }) => {
    const source$ = interval(20, testScheduler).pipe(take(5));
    const expected = '--a-b-c-d-(e|)';
    const values = { a: 0, b: 1, c: 2, d: 3, e: 4 };

    expectObservable(source$).toBe(expected, values);
  });
});
```

### Testing timeout

```typescript
import { timeout } from 'rxjs/operators';

it('should timeout if no emission within timeframe', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('----------a--|');
    const expected =     '-----#';

    const result$ = source$.pipe(
      timeout(50) // 5 frames
    );

    expectObservable(result$).toBe(expected);
  });
});
```

## Testing Error Scenarios

### Testing catchError

```typescript
import { catchError, of } from 'rxjs';

it('should catch and recover from errors', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--#', { a: 1, b: 2 });
    const expected =     '--a--b--(c|)';
    const values = { a: 1, b: 2, c: 0 };

    const result$ = source$.pipe(
      catchError(() => of(0))
    );

    expectObservable(result$).toBe(expected, values);
  });
});
```

### Testing retry

```typescript
import { retry } from 'rxjs/operators';

it('should retry on error', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--#');
    const expected =     '--a--b----a--b----a--b----a--b--#';

    const result$ = source$.pipe(
      retry(3) // 3 retries after initial attempt
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing retryWhen

```typescript
import { retryWhen, delay, take } from 'rxjs/operators';

it('should retry with delay', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--#');
    const expected =     '--#--#--#--(#|)';

    const result$ = source$.pipe(
      retryWhen(errors => errors.pipe(
        delay(20),
        take(3)
      ))
    );

    expectObservable(result$).toBe(expected);
  });
});
```

## Testing Angular Services with Marble Testing

### Testing Search Service with Debounce

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class SearchService {
  constructor(private http: HttpClient) {}

  search(terms$: Observable<string>): Observable<any[]> {
    return terms$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => this.http.get<any[]>(`/api/search?q=${term}`))
    );
  }
}

// Test
describe('SearchService', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce search terms', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const terms$ = cold('a-b-c----d------|', {
        a: 'ang',
        b: 'angu',
        c: 'angul',
        d: 'angular'
      });

      // Only 'angul' and 'angular' should trigger search after debounce
      const expected = '----c-------d---|';

      const result$ = terms$.pipe(
        debounceTime(30),
        distinctUntilChanged()
      );

      expectObservable(result$).toBe(expected, {
        c: 'angul',
        d: 'angular'
      });
    });
  });
});
```

### Testing Data Service with Cache

```typescript
@Injectable({
  providedIn: 'root'
})
export class DataService {
  private cache$ = new BehaviorSubject<any[]>([]);

  constructor(private http: HttpClient) {}

  getData(refresh = false): Observable<any[]> {
    if (!refresh && this.cache$.value.length > 0) {
      return this.cache$.asObservable();
    }

    return this.http.get<any[]>('/api/data').pipe(
      tap(data => this.cache$.next(data)),
      shareReplay(1)
    );
  }
}

// Test
it('should return cached data on subsequent calls', () => {
  testScheduler.run(({ cold, hot, expectObservable }) => {
    // Mock HTTP response
    const httpResponse$ = cold('---a|', { a: [1, 2, 3] });

    // First call - hits API
    const first$ = hot('^--!', null);
    const expected1 = '---a';

    // Second call - returns cache immediately
    const second$ = hot('-----^!', null);
    const expected2 = '-----a';

    // Tests would check cache behavior
  });
});
```

### Testing Reactive Form Validation

```typescript
import { FormControl } from '@angular/forms';
import { debounceTime, map } from 'rxjs/operators';

function validateUsername(control: FormControl): Observable<any> {
  return control.valueChanges.pipe(
    debounceTime(300),
    map(value => {
      return value.length < 3 ? { minLength: true } : null;
    })
  );
}

// Test
it('should validate username with debounce', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const values$ = cold('a-b-c----d------|', {
      a: 'j',
      b: 'jo',
      c: 'joh',
      d: 'john'
    });

    const expected = '----c-------d---|';
    const expectedValues = {
      c: null,        // Valid (>= 3 chars)
      d: null         // Valid (>= 3 chars)
    };

    const result$ = values$.pipe(
      debounceTime(30),
      map(value => value.length < 3 ? { minLength: true } : null)
    );

    expectObservable(result$).toBe(expected, expectedValues);
  });
});
```

## Advanced Marble Testing Patterns

### Testing combineLatest

```typescript
import { combineLatest } from 'rxjs';

it('should combine latest values', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const a$ = cold('--a-----c--|', { a: 1, c: 3 });
    const b$ = cold('----b-----d--|', { b: 2, d: 4 });
    const expected = '----x---y-z-|';
    const values = {
      x: [1, 2],
      y: [3, 2],
      z: [3, 4]
    };

    const result$ = combineLatest([a$, b$]);

    expectObservable(result$).toBe(expected, values);
  });
});
```

### Testing merge

```typescript
import { merge } from 'rxjs';

it('should merge multiple observables', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const a$ = cold('--a-----c--|');
    const b$ = cold('----b-----d--|');
    const expected = '--a-b---c-d--|';

    const result$ = merge(a$, b$);

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing switchMap

```typescript
import { switchMap } from 'rxjs/operators';

it('should switch to new inner observable', () => {
  testScheduler.run(({ cold, hot, expectObservable }) => {
    const outer$ = hot('--a-----b------|');
    const inner1$ = cold('  --1--2--3--|');
    const inner2$ = cold('        --4--5--6--|');
    const expected =     '----1---4--5--6--|';

    const result$ = outer$.pipe(
      switchMap(val => val === 'a' ? inner1$ : inner2$)
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing concatMap

```typescript
import { concatMap } from 'rxjs/operators';

it('should concat inner observables in sequence', () => {
  testScheduler.run(({ cold, hot, expectObservable }) => {
    const outer$ = hot('--a--b--|');
    const inner$ = cold('  --x--y|');
    const expected =     '----x--y--x--y|';

    const result$ = outer$.pipe(
      concatMap(() => inner$)
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing forkJoin

```typescript
import { forkJoin } from 'rxjs';

it('should emit when all observables complete', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const a$ = cold('--a--|', { a: 1 });
    const b$ = cold('----b--|', { b: 2 });
    const c$ = cold('------c--|', { c: 3 });
    const expected = '-------(x|)';
    const values = { x: [1, 2, 3] };

    const result$ = forkJoin([a$, b$, c$]);

    expectObservable(result$).toBe(expected, values);
  });
});
```

### Testing with Subscription Timing

```typescript
it('should test subscription timing', () => {
  testScheduler.run(({ cold, expectObservable, expectSubscriptions }) => {
    const source$ = cold('--a--b--c--|');
    const subs =         '^----------!';
    const expected =     '--a--b--c--|';

    expectObservable(source$).toBe(expected);
    expectSubscriptions(source$.subscriptions).toBe(subs);
  });
});

it('should test early unsubscription', () => {
  testScheduler.run(({ cold, expectObservable, expectSubscriptions }) => {
    const source$ = cold('--a--b--c--|');
    const unsub =        '------!';
    const subs =         '^-----!';
    const expected =     '--a--b-';

    expectObservable(source$, unsub).toBe(expected);
    expectSubscriptions(source$.subscriptions).toBe(subs);
  });
});
```

### Testing takeUntil

```typescript
import { takeUntil } from 'rxjs/operators';

it('should complete when notifier emits', () => {
  testScheduler.run(({ cold, hot, expectObservable }) => {
    const source$ = cold('--a--b--c--d--e--|');
    const notifier$ = hot('--------x--------|');
    const expected =     '--a--b--c';
    const sub =          '^-------!';

    const result$ = source$.pipe(
      takeUntil(notifier$)
    );

    expectObservable(result$, sub).toBe(expected);
  });
});
```

## Common Mistakes

### 1. Forgetting to Run TestScheduler

```typescript
// BAD: Not wrapping in testScheduler.run()
it('incorrect test', () => {
  const source$ = cold('--a--b--|'); // Error: cold is not defined
});

// GOOD: Properly wrapped
it('correct test', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--|');
    expectObservable(source$).toBe('--a--b--|');
  });
});
```

### 2. Incorrect Time Frame Counting

```typescript
// BAD: Miscounting frames
it('incorrect timing', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--|');
    const expected =     '--a--|'; // Should be at same position
    
    const result$ = source$.pipe(delay(10)); // 1 frame
    expectObservable(result$).toBe('--a--|'); // Wrong! Should be '---a--|'
  });
});

// GOOD: Correct frame counting
it('correct timing', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--|');
    const expected =     '---a--|'; // Correctly delayed by 1 frame
    
    const result$ = source$.pipe(delay(10));
    expectObservable(result$).toBe(expected);
  });
});
```

### 3. Not Providing Value Dictionaries

```typescript
// BAD: No value mapping
it('missing values', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--|'); // What is 'a'?
    expectObservable(source$).toBe('--a--|'); // Unclear what 'a' represents
  });
});

// GOOD: Clear value mapping
it('with values', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--|', { a: { id: 1, name: 'John' } });
    const expected =     '--a--|';
    const values = { a: { id: 1, name: 'John' } };
    
    expectObservable(source$).toBe(expected, values);
  });
});
```

### 4. Using Real Time Operations

```typescript
// BAD: Using real setTimeout
it('real time - slow test', (done) => {
  setTimeout(() => {
    expect(true).toBe(true);
    done();
  }, 1000); // Actually waits 1 second!
});

// GOOD: Virtual time
it('virtual time - fast test', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--|');
    expectObservable(source$).toBe('--a--|');
  }); // Completes instantly
});
```

### 5. Mixing Hot and Cold Incorrectly

```typescript
// BAD: Using cold when hot is needed
it('incorrect observable type', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    // Testing a Subject but using cold
    const subject = new Subject();
    const source$ = cold('--a--b--|'); // This doesn't represent Subject behavior
  });
});

// GOOD: Using hot for Subject-like behavior
it('correct observable type', () => {
  testScheduler.run(({ hot, expectObservable }) => {
    const source$ = hot('--a--b--|'); // Correctly represents hot observable
    expectObservable(source$).toBe('--a--b--|');
  });
});
```

## Best Practices

### 1. Use Descriptive Marble Diagrams

```typescript
// GOOD: Clear and readable
it('should debounce user input', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    // Rapid typing followed by pause
    const input$ =   'a-b-c----d------|';
    const expected = '----c-------d---|';
    // Only emit after 3 frames of silence
  });
});
```

### 2. Test Both Timing and Values

```typescript
it('should transform and delay', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--|', { a: 1, b: 2 });
    const expected =     '----a--b--|';
    const values = { a: 10, b: 20 };

    const result$ = source$.pipe(
      map(x => x * 10),
      delay(20)
    );

    expectObservable(result$).toBe(expected, values);
  });
});
```

### 3. Break Complex Tests into Smaller Ones

```typescript
// GOOD: Separate concerns
describe('Complex Observable Pipeline', () => {
  it('should debounce input', () => {
    // Test only debounce behavior
  });

  it('should filter empty values', () => {
    // Test only filtering
  });

  it('should map to API calls', () => {
    // Test only mapping
  });

  it('should handle the complete pipeline', () => {
    // Integration test
  });
});
```

### 4. Use Consistent Time Frames

```typescript
// GOOD: Consistent spacing for readability
const source1$ = cold('--a--b--c--|');
const source2$ = cold('----x--y--z--|');
const expected =      '--a-xb-yc-z--|';
```

### 5. Test Error Recovery

```typescript
it('should handle errors gracefully', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--#', { a: 1 });
    const expected =     '--a--(b|)';
    const values = { a: 1, b: 'error' };

    const result$ = source$.pipe(
      catchError(() => of('error'))
    );

    expectObservable(result$).toBe(expected, values);
  });
});
```

## Interview Questions

### 1. What is marble testing and why is it useful?

**Answer:** Marble testing is a technique for testing RxJS observables using ASCII-based marble diagrams that represent observable streams over time. It's useful because:

- **Synchronous testing**: Virtualizes time, making async tests run synchronously and fast
- **Visual clarity**: Marble diagrams provide intuitive visual representation of observable behavior
- **Timing precision**: Easily test timing-based operators like debounce, throttle, and delay
- **Deterministic**: Tests are reliable and don't depend on real-world timing
- **Comprehensive**: Can test hot/cold observables, errors, completions, and subscriptions

### 2. Explain the difference between hot and cold observables in marble testing.

**Answer:**

**Cold Observables**: Created with `cold()`, start emitting when subscribed, each subscription gets independent stream starting from beginning.

```typescript
const cold$ = cold('--a--b--|'); // Each subscriber gets a, b
```

**Hot Observables**: Created with `hot()`, already emitting regardless of subscriptions, subscribers only get emissions after their subscription point (marked with `^`).

```typescript
const hot$ = hot('--a--b--^-c--d--|');
// Subscribers starting at ^ only get c, d (miss a, b)
```

### 3. What does each symbol mean in marble diagrams?

**Answer:**
- `-`: Time frame (10ms by default)
- `|`: Completion
- `#`: Error
- `a`, `b`, `c`: Emitted values
- `(ab)`: Synchronous grouped emissions
- `^`: Subscription point (hot observables)
- `!`: Unsubscription point
- `' '`: Space for readability (ignored)

### 4. How do you test time-based operators like debounceTime?

**Answer:** Use marble diagrams with proper frame spacing to represent the debounce period:

```typescript
it('should debounce', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('a-b-c----d------|');
    const expected =     '----c-------d---|';
    // 3 frames silence required (30ms)
    
    const result$ = source$.pipe(debounceTime(30));
    expectObservable(result$).toBe(expected);
  });
});
```

### 5. How do you test error scenarios with marble testing?

**Answer:** Use `#` to represent errors in marble diagrams:

```typescript
it('should catch errors', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a--b--#', { a: 1, b: 2 });
    const expected =     '--a--b--(c|)';
    const values = { a: 1, b: 2, c: 0 };
    
    const result$ = source$.pipe(
      catchError(() => of(0))
    );
    
    expectObservable(result$).toBe(expected, values);
  });
});
```

### 6. What is TestScheduler and what does its run method do?

**Answer:** `TestScheduler` is RxJS's virtual time scheduler for marble testing. The `run()` method:

- Creates a virtual time context where time is controllable
- Provides helper functions (`cold`, `hot`, `expectObservable`, etc.)
- Automatically advances virtual time to process all scheduled operations
- Synchronously executes all observable emissions for fast tests
- Compares actual vs expected behavior using provided assertion function

### 7. How do you test subscription timing?

**Answer:** Use `expectSubscriptions()` with subscription marble diagrams:

```typescript
it('should track subscription timing', () => {
  testScheduler.run(({ cold, expectObservable, expectSubscriptions }) => {
    const source$ = cold('--a--b--c--|');
    const subs =         '^----------!';
    const unsub =        '------!';
    
    expectObservable(source$, unsub).toBe('--a--b-');
    expectSubscriptions(source$.subscriptions).toBe('^-----!');
  });
});
```

### 8. How do you test operators like combineLatest or merge?

**Answer:** Create multiple source observables and verify the combined output:

```typescript
it('should combine latest values', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const a$ = cold('--a-----c--|', { a: 1, c: 3 });
    const b$ = cold('----b-----d--|', { b: 2, d: 4 });
    const expected = '----x---y-z-|';
    const values = {
      x: [1, 2], // First emission when both have emitted
      y: [3, 2], // a updates to c
      z: [3, 4]  // b updates to d
    };
    
    expectObservable(combineLatest([a$, b$])).toBe(expected, values);
  });
});
```

### 9. What are common mistakes in marble testing?

**Answer:**
- Forgetting to wrap tests in `testScheduler.run()`
- Miscounting time frames in diagrams
- Not providing value dictionaries for assertions
- Using real time operations instead of virtual time
- Mixing hot and cold observables incorrectly
- Not testing subscription/unsubscription timing
- Testing implementation instead of behavior

### 10. How does marble testing improve test performance?

**Answer:** Marble testing dramatically improves performance by:

- **Virtual time**: No actual waiting for timeouts, delays, or intervals
- **Synchronous execution**: Async operations run synchronously in virtual time
- **Instant completion**: Tests that would take seconds/minutes complete in milliseconds
- **No race conditions**: Deterministic timing eliminates flakiness
- **Efficient scheduling**: TestScheduler batches and optimizes all scheduled operations

Example: A test with `debounceTime(1000)` completes instantly instead of waiting 1+ seconds.

## Key Takeaways

1. **Marble testing uses ASCII diagrams** to represent observable streams, making complex async behavior visual and testable
2. **TestScheduler virtualizes time**, allowing synchronous testing of time-based operations without actual delays
3. **Cold observables** start emitting on subscription; hot observables emit regardless of subscribers
4. **Each dash represents 10ms** by default; use consistent spacing for readability
5. **Use symbols** `|` for completion, `#` for errors, `()` for synchronous emissions, `^` for subscription points
6. **Always wrap tests** in `testScheduler.run()` to access marble testing helpers
7. **Provide value dictionaries** when testing with complex data structures for clear assertions
8. **Test timing and values separately** when operators transform both
9. **Marble testing makes tests fast and deterministic** by eliminating real-world timing dependencies
10. **Use expectSubscriptions()** to verify subscription and unsubscription timing behavior

## Resources

### Official Documentation
- [RxJS Marble Testing](https://rxjs.dev/guide/testing/marble-testing)
- [RxJS TestScheduler API](https://rxjs.dev/api/testing/TestScheduler)
- [RxJS Operators](https://rxjs.dev/api)

### Tools and Libraries
- [rxjs-marbles](https://github.com/cartant/rxjs-marbles) - Marble testing utilities
- [jasmine-marbles](https://github.com/synapse-wireless-labs/jasmine-marbles) - Jasmine matchers for marble testing
- [jest-marbles](https://github.com/just-jeb/jest-marbles) - Jest utilities for marble testing

### Interactive Learning
- [RxViz](https://rxviz.com/) - Visualize RxJS observables
- [RxJS Marbles](https://rxmarbles.com/) - Interactive marble diagrams
- [Learn RxJS](https://www.learnrxjs.io/) - RxJS patterns and examples

### Articles and Tutorials
- [Understanding Marble Diagrams](https://medium.com/@bencabanes/marble-testing-observable-introduction-1f5ad39231c)
- [Testing RxJS with Marble Diagrams](https://blog.angularindepth.com/how-to-test-observables-a00038c7faad)

### Video Tutorials
- [Marble Testing in RxJS](https://www.youtube.com/watch?v=s7D57f2y9YI)
- [Advanced RxJS Testing](https://www.youtube.com/watch?v=CFGvzcn8WHk)

### Books
- "RxJS in Action" by Paul P. Daniels and Luis Atencio
- "Reactive Programming with RxJS" by Sergi Mansilla
