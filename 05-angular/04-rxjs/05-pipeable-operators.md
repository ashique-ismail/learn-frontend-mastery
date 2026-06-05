# Pipeable Operators in RxJS

## The Idea

**In plain English:** Pipeable operators are steps you connect together to process a stream of data — each step takes the data in, does one thing to it (like filtering out items you do not want, or changing their shape), and passes the result to the next step. Think of them as a series of stations on an assembly line, where each station transforms the product before sending it along.

**Real-world analogy:** Imagine a juice factory where fruit arrives on a conveyor belt. Workers at each station do one job: the first station removes rotten fruit, the second squeezes the fruit into juice, and the third fills only bottles that meet the right size.
- The conveyor belt = the Observable (the stream of data flowing through)
- Each worker station = a pipeable operator (one focused transformation step)
- The finished bottled juice at the end = the final value your code receives after all operators have run

---

## Table of Contents
- [Introduction](#introduction)
- [The pipe() Method](#the-pipe-method)
- [Operator Composition](#operator-composition)
- [Common Transformation Operators](#common-transformation-operators)
- [Custom Operators](#custom-operators)
- [Operator Chaining](#operator-chaining)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Pipeable operators are pure functions that take an Observable as input and return a new Observable as output. They enable functional composition and transformation of data streams without modifying the original Observable.

Understanding pipeable operators is crucial for:
- Transforming data streams
- Building complex reactive logic
- Creating reusable operator chains
- Maintaining clean, functional code

## The pipe() Method

The `pipe()` method chains operators together:

```typescript
import { of } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// Basic pipe usage
const source$ = of(1, 2, 3, 4, 5);

const result$ = source$.pipe(
  filter(n => n % 2 === 0),  // Keep even numbers
  map(n => n * 10)            // Multiply by 10
);

result$.subscribe(val => console.log(val));
// Output: 20, 40
```

### Multiple Operators

```typescript
import { of } from 'rxjs';
import { map, filter, tap, take } from 'rxjs/operators';

const numbers$ = of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

numbers$.pipe(
  tap(n => console.log('Original:', n)),
  filter(n => n > 3),
  tap(n => console.log('After filter:', n)),
  map(n => n * 2),
  tap(n => console.log('After map:', n)),
  take(3)
).subscribe(result => console.log('Final:', result));
```

### Pipe in Angular Components

```typescript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { map, catchError, tap } from 'rxjs/operators';
import { of } from 'rxjs';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }} ({{ user.email }})
    </div>
  `
})
export class UserListComponent implements OnInit {
  users$ = this.http.get<User[]>('/api/users').pipe(
    tap(users => console.log('Received users:', users.length)),
    map(users => users.filter(u => u.email.includes('@company.com'))),
    catchError(error => {
      console.error('Error loading users:', error);
      return of([]);
    })
  );
  
  constructor(private http: HttpClient) {}
  
  ngOnInit() {}
}
```

## Operator Composition

Operators can be composed and reused:

```typescript
import { pipe } from 'rxjs';
import { map, filter, tap } from 'rxjs/operators';

// Create reusable operator combination
const processNumbers = pipe(
  tap(n => console.log('Input:', n)),
  filter((n: number) => n > 0),
  map((n: number) => n * 2),
  tap(n => console.log('Output:', n))
);

// Use the composed operator
const source1$ = of(-1, 0, 1, 2, 3);
source1$.pipe(processNumbers).subscribe();

const source2$ = of(5, 6, 7);
source2$.pipe(processNumbers).subscribe();
```

### Composed Operators as Functions

```typescript
import { Observable, pipe } from 'rxjs';
import { map, filter, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

// Reusable operator function
function filterAndDouble<T extends number>() {
  return pipe(
    filter((n: T) => n > 0),
    map((n: T) => n * 2)
  );
}

// Reusable error handling
function handleHttpError<T>(fallbackValue: T) {
  return pipe(
    catchError(error => {
      console.error('HTTP Error:', error);
      return of(fallbackValue);
    })
  );
}

// Usage
@Component({
  selector: 'app-data',
  template: `<div>{{ data$ | async | json }}</div>`
})
export class DataComponent {
  data$ = this.http.get<any[]>('/api/data').pipe(
    handleHttpError([])
  );
  
  constructor(private http: HttpClient) {}
}
```

### Operator Factory Pattern

```typescript
import { Observable, OperatorFunction } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// Operator factory
function filterByProperty<T>(
  property: keyof T,
  value: any
): OperatorFunction<T, T> {
  return filter(item => item[property] === value);
}

function mapToProperty<T, K extends keyof T>(
  property: K
): OperatorFunction<T, T[K]> {
  return map(item => item[property]);
}

// Usage
interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
}

@Component({
  selector: 'app-products',
  template: `
    <div *ngFor="let name of productNames$ | async">
      {{ name }}
    </div>
  `
})
export class ProductsComponent {
  productNames$ = this.http.get<Product[]>('/api/products').pipe(
    filterByProperty('category', 'electronics'),
    mapToProperty('name')
  );
  
  constructor(private http: HttpClient) {}
}
```

## Common Transformation Operators

### map Operator

```typescript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

// Simple transformation
of(1, 2, 3).pipe(
  map(n => n * 10)
).subscribe(val => console.log(val));
// Output: 10, 20, 30

// Object transformation
interface User {
  firstName: string;
  lastName: string;
}

of<User>(
  { firstName: 'John', lastName: 'Doe' },
  { firstName: 'Jane', lastName: 'Smith' }
).pipe(
  map(user => `${user.firstName} ${user.lastName}`)
).subscribe(name => console.log(name));
// Output: 'John Doe', 'Jane Smith'
```

### pluck Operator (deprecated - use map)

```typescript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

interface User {
  id: number;
  profile: {
    name: string;
    email: string;
  };
}

const users$ = of<User>(
  { id: 1, profile: { name: 'John', email: 'john@example.com' } },
  { id: 2, profile: { name: 'Jane', email: 'jane@example.com' } }
);

// Extract nested property
users$.pipe(
  map(user => user.profile.name)
).subscribe(name => console.log(name));
// Output: 'John', 'Jane'
```

### scan Operator

```typescript
import { of } from 'rxjs';
import { scan } from 'rxjs/operators';

// Accumulator (like Array.reduce)
of(1, 2, 3, 4, 5).pipe(
  scan((acc, curr) => acc + curr, 0)
).subscribe(val => console.log(val));
// Output: 1, 3, 6, 10, 15

// Running count
@Component({
  selector: 'app-counter',
  template: `
    <button (click)="increment()">+</button>
    <button (click)="decrement()">-</button>
    <div>Count: {{ count$ | async }}</div>
  `
})
export class CounterComponent {
  private actionSubject = new Subject<number>();
  
  count$ = this.actionSubject.pipe(
    scan((count, delta) => count + delta, 0)
  );
  
  increment() {
    this.actionSubject.next(1);
  }
  
  decrement() {
    this.actionSubject.next(-1);
  }
}
```

### reduce Operator

```typescript
import { of } from 'rxjs';
import { reduce } from 'rxjs/operators';

// Like scan but only emits final result
of(1, 2, 3, 4, 5).pipe(
  reduce((acc, curr) => acc + curr, 0)
).subscribe(val => console.log(val));
// Output: 15 (only final sum)

// Calculate average
of(10, 20, 30, 40, 50).pipe(
  reduce(
    (acc, val, index) => {
      acc.sum += val;
      acc.count = index + 1;
      return acc;
    },
    { sum: 0, count: 0 }
  ),
  map(({ sum, count }) => sum / count)
).subscribe(avg => console.log('Average:', avg));
// Output: Average: 30
```

## Custom Operators

### Creating Custom Operators

```typescript
import { Observable, OperatorFunction, pipe } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// Custom operator to double numbers
function double<T extends number>(): OperatorFunction<T, T> {
  return map(value => (value * 2) as T);
}

// Custom operator with parameters
function multiplyBy<T extends number>(
  multiplier: number
): OperatorFunction<T, T> {
  return map(value => (value * multiplier) as T);
}

// Usage
of(1, 2, 3).pipe(
  double()
).subscribe(val => console.log(val));
// Output: 2, 4, 6

of(1, 2, 3).pipe(
  multiplyBy(5)
).subscribe(val => console.log(val));
// Output: 5, 10, 15
```

### Complex Custom Operators

```typescript
import { Observable, OperatorFunction } from 'rxjs';
import { map, scan, distinctUntilChanged } from 'rxjs/operators';

// Custom operator: calculate moving average
function movingAverage(
  windowSize: number
): OperatorFunction<number, number> {
  return (source: Observable<number>) => {
    return source.pipe(
      scan(
        (acc, value) => {
          acc.values.push(value);
          if (acc.values.length > windowSize) {
            acc.values.shift();
          }
          return acc;
        },
        { values: [] as number[] }
      ),
      map(acc => {
        const sum = acc.values.reduce((s, v) => s + v, 0);
        return sum / acc.values.length;
      })
    );
  };
}

// Usage
of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10).pipe(
  movingAverage(3)
).subscribe(avg => console.log('Moving avg:', avg));
```

### Logging Operator

```typescript
import { Observable, OperatorFunction } from 'rxjs';
import { tap } from 'rxjs/operators';

// Custom debug operator
function debug<T>(tag: string): OperatorFunction<T, T> {
  return tap({
    next: value => console.log(`[${tag}] Next:`, value),
    error: error => console.error(`[${tag}] Error:`, error),
    complete: () => console.log(`[${tag}] Complete`)
  });
}

// Usage
of(1, 2, 3).pipe(
  debug('Source'),
  map(n => n * 2),
  debug('After map'),
  filter(n => n > 2),
  debug('After filter')
).subscribe();
```

### Retry with Delay

```typescript
import { Observable, OperatorFunction, throwError, timer } from 'rxjs';
import { mergeMap, finalize, retryWhen } from 'rxjs/operators';

function retryWithDelay<T>(
  delayMs: number,
  maxRetries: number = 3
): OperatorFunction<T, T> {
  return (source: Observable<T>) => {
    return source.pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error, index) => {
            const retryAttempt = index + 1;
            
            if (retryAttempt > maxRetries) {
              return throwError(() => error);
            }
            
            console.log(
              `Retry attempt ${retryAttempt}/${maxRetries} after ${delayMs}ms`
            );
            
            return timer(delayMs);
          })
        )
      )
    );
  };
}

// Usage
@Component({
  selector: 'app-retry-demo',
  template: `<div>{{ data$ | async | json }}</div>`
})
export class RetryDemoComponent {
  data$ = this.http.get('/api/data').pipe(
    retryWithDelay(1000, 3),
    catchError(error => {
      console.error('All retries failed:', error);
      return of({ error: 'Failed to load data' });
    })
  );
  
  constructor(private http: HttpClient) {}
}
```

## Operator Chaining

### Building Complex Pipelines

```typescript
import { Component } from '@angular/core';
import { FormControl } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap, map, catchError, startWith } from 'rxjs/operators';
import { of } from 'rxjs';

@Component({
  selector: 'app-search',
  template: `
    <input [formControl]="searchControl" placeholder="Search...">
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngFor="let result of results$ | async">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent {
  searchControl = new FormControl('');
  
  results$ = this.searchControl.valueChanges.pipe(
    startWith(''),
    debounceTime(300),
    distinctUntilChanged(),
    map(term => term.trim()),
    switchMap(term => {
      if (term.length < 2) {
        return of([]);
      }
      return this.http.get<any[]>(`/api/search?q=${term}`).pipe(
        catchError(() => of([]))
      );
    })
  );
  
  loading$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    map(() => true),
    switchMap(() => this.results$.pipe(map(() => false)))
  );
  
  constructor(private http: HttpClient) {}
}
```

### Conditional Operator Application

```typescript
import { Observable } from 'rxjs';

function conditionalOperator<T>(
  condition: boolean,
  operator: OperatorFunction<T, T>
): OperatorFunction<T, T> {
  return (source: Observable<T>) => {
    return condition ? source.pipe(operator) : source;
  };
}

// Usage
const source$ = of(1, 2, 3, 4, 5);

const DEBUG = true;

source$.pipe(
  map(n => n * 2),
  conditionalOperator(DEBUG, debug('After map')),
  filter(n => n > 5),
  conditionalOperator(DEBUG, debug('After filter'))
).subscribe();
```

## Common Mistakes

### Mistake 1: Not Returning in pipe

```typescript
// BAD: Not returning from operator
of(1, 2, 3).pipe(
  map(n => {
    n * 2; // Missing return!
  })
).subscribe(val => console.log(val));
// Output: undefined, undefined, undefined

// GOOD: Return the value
of(1, 2, 3).pipe(
  map(n => n * 2)
).subscribe(val => console.log(val));
// Output: 2, 4, 6
```

### Mistake 2: Nested Subscriptions

```typescript
// BAD: Nested subscriptions
this.userService.getUser().subscribe(user => {
  this.postService.getPosts(user.id).subscribe(posts => {
    // Nested subscription hell
  });
});

// GOOD: Use switchMap
this.userService.getUser().pipe(
  switchMap(user => this.postService.getPosts(user.id))
).subscribe(posts => {
  // Clean, flat subscription
});
```

### Mistake 3: Modifying Values Instead of Returning New

```typescript
// BAD: Mutating objects
interface User {
  name: string;
  age: number;
}

of({ name: 'John', age: 30 }).pipe(
  map(user => {
    user.age++; // Mutation!
    return user;
  })
).subscribe();

// GOOD: Return new object
of({ name: 'John', age: 30 }).pipe(
  map(user => ({
    ...user,
    age: user.age + 1
  }))
).subscribe();
```

## Best Practices

### 1. Keep Operators Pure

```typescript
// Pure operators - no side effects
const pure$ = source$.pipe(
  map(x => x * 2),
  filter(x => x > 10)
);

// Side effects in tap only
const withSideEffects$ = source$.pipe(
  tap(x => console.log('Value:', x)),
  map(x => x * 2),
  tap(x => this.analyticsService.track(x))
);
```

### 2. Compose Reusable Operators

```typescript
// Create reusable operator combinations
const handleApiResponse = <T>() => pipe(
  map((response: any) => response.data as T),
  catchError(error => {
    console.error('API Error:', error);
    return of(null);
  })
);

// Use across multiple services
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers() {
    return this.http.get('/api/users').pipe(
      handleApiResponse<User[]>()
    );
  }
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  getProducts() {
    return this.http.get('/api/products').pipe(
      handleApiResponse<Product[]>()
    );
  }
}
```

### 3. Type Your Operators

```typescript
import { OperatorFunction } from 'rxjs';
import { map } from 'rxjs/operators';

function toUpperCase(): OperatorFunction<string, string> {
  return map(str => str.toUpperCase());
}

function extractProperty<T, K extends keyof T>(
  key: K
): OperatorFunction<T, T[K]> {
  return map(obj => obj[key]);
}

// Type-safe usage
of('hello').pipe(
  toUpperCase()
).subscribe(val => console.log(val)); // val is string

interface User {
  name: string;
  age: number;
}

of({ name: 'John', age: 30 }).pipe(
  extractProperty('name')
).subscribe(name => console.log(name)); // name is string
```

### 4. Use Descriptive Operator Names

```typescript
// Custom operators with clear names
function filterActive<T extends { active: boolean }>(): OperatorFunction<T, T> {
  return filter(item => item.active);
}

function sortByDate<T extends { date: Date }>(): OperatorFunction<T[], T[]> {
  return map(items => 
    [...items].sort((a, b) => b.date.getTime() - a.date.getTime())
  );
}

// Clear, readable pipeline
products$.pipe(
  filterActive(),
  sortByDate(),
  take(10)
).subscribe();
```

## Interview Questions

### Q1: What are pipeable operators?
**Answer:** Pipeable operators are pure functions that take an Observable as input and return a new Observable as output. They enable functional composition through the pipe() method and don't modify the source Observable. Examples include map, filter, switchMap, etc.

### Q2: Why use pipe() instead of chaining methods?
**Answer:** The pipe() method enables tree-shaking (smaller bundles), better TypeScript type inference, and easier creation of custom operators. It's the modern approach that replaced the old "dot-chaining" syntax (source.map().filter()).

### Q3: How do you create a custom operator?
**Answer:** Create a function that returns an OperatorFunction. This function takes a source Observable and returns a new Observable. You can use existing operators inside or create completely custom logic. The operator should be pure and not modify the source.

### Q4: What's the difference between map and tap?
**Answer:** map transforms values and returns a new value for the stream. tap performs side effects (like logging) but returns the original value unchanged. Use map for transformations, tap for debugging, logging, or other side effects.

### Q5: Can operators be reused?
**Answer:** Yes! Operators are just functions, so you can create reusable operator combinations using pipe() or create custom operators. This promotes DRY principles and makes code more maintainable.

## Key Takeaways

1. Pipeable operators are pure functions that transform Observables
2. Use pipe() method to chain multiple operators together
3. Operators don't modify the source Observable (immutable)
4. Create custom operators for reusable logic
5. Keep operators pure - side effects only in tap
6. Type your custom operators for type safety
7. Compose operators for complex transformations
8. Use map for transformation, tap for side effects
9. Avoid nested subscriptions - use flattening operators
10. Name custom operators descriptively

## Resources

### Official Documentation
- [RxJS Operators Guide](https://rxjs.dev/guide/operators)
- [Pipeable Operators](https://rxjs.dev/guide/operators#piping)
- [Creating Custom Operators](https://rxjs.dev/guide/operators#creating-new-operators-from-scratch)

### Articles
- [Understanding Pipeable Operators](https://angular.io/guide/rx-library)
- [Custom RxJS Operators](https://blog.angular.io/rxjs-custom-operators)

### Tools
- RxJS Marbles for visualization
- TypeScript for type checking
