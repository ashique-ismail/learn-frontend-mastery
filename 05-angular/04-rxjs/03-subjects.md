# Subjects in RxJS

## The Idea

**In plain English:** A Subject is a special object in RxJS that works like a live radio station — it can broadcast (send out) data values at any time, and multiple listeners can tune in to receive those values at the same moment. Unlike a regular Observable (which is more like a pre-recorded podcast), a Subject lets you push new values into it whenever you want.

**Real-world analogy:** Imagine a school PA (public address) system. The principal speaks into the microphone, and every classroom that has the speaker turned on hears the announcement at the same time. If your classroom speaker is off when the announcement plays, you miss it entirely.

- The principal speaking into the microphone = calling `subject.next(value)` to push a new value
- The PA system's broadcast channel = the Subject itself, which holds no history and multicasts to all listeners
- Each classroom with the speaker on = a subscriber listening to the Subject's observable stream

---

## Table of Contents

- [Introduction](#introduction)
- [Subject Basics](#subject-basics)
- [BehaviorSubject](#behaviorsubject)
- [ReplaySubject](#replaysubject)
- [AsyncSubject](#asyncsubject)
- [Subject vs Observable](#subject-vs-observable)
- [Use Cases](#use-cases)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Subjects are special types of Observables that act as both Observable and Observer. They can multicast values to multiple subscribers and you can push values into them. Subjects are essential for event buses, state management, and sharing data between components.

Understanding Subjects is crucial for:

- Implementing service-based state management
- Creating event buses
- Sharing data between components
- Converting cold observables to hot

## Subject Basics

A Subject is both an Observable and an Observer:

```typescript
import { Subject } from 'rxjs';

// Create a subject
const subject$ = new Subject<number>();

// Subscribe as Observable
subject$.subscribe({
  next: (value) => console.log('Observer A:', value)
});

subject$.subscribe({
  next: (value) => console.log('Observer B:', value)
});

// Push values as Observer
subject$.next(1);
subject$.next(2);

// Output:
// Observer A: 1
// Observer B: 1
// Observer A: 2
// Observer B: 2
```

### Subject as Multicast

```typescript
import { Subject, interval } from 'rxjs';
import { take } from 'rxjs/operators';

// Cold observable (unicast)
const cold$ = interval(1000).pipe(take(3));

// Without Subject - each subscriber gets own execution
cold$.subscribe(val => console.log('Sub A:', val));
cold$.subscribe(val => console.log('Sub B:', val));
// Both get: 0, 1, 2 at different times

// With Subject - shared execution (multicast)
const subject$ = new Subject<number>();
const hot$ = interval(1000).pipe(take(3));

hot$.subscribe(subject$);

subject$.subscribe(val => console.log('Sub A:', val));
subject$.subscribe(val => console.log('Sub B:', val));
// Both get same values at same time
```

### Subject in Angular Service

```typescript
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';

interface Message {
  text: string;
  type: 'info' | 'warning' | 'error';
}

@Injectable({
  providedIn: 'root'
})
export class NotificationService {
  private messageSubject = new Subject<Message>();
  
  // Expose as Observable (read-only)
  messages$: Observable<Message> = this.messageSubject.asObservable();
  
  // Method to push new messages
  showMessage(text: string, type: Message['type'] = 'info') {
    this.messageSubject.next({ text, type });
  }
  
  showInfo(text: string) {
    this.showMessage(text, 'info');
  }
  
  showWarning(text: string) {
    this.showMessage(text, 'warning');
  }
  
  showError(text: string) {
    this.showMessage(text, 'error');
  }
}

// Component usage
@Component({
  selector: 'app-notification',
  template: `
    <div *ngFor="let msg of messages$ | async" 
         [class]="msg.type">
      {{ msg.text }}
    </div>
  `
})
export class NotificationComponent {
  messages$ = this.notificationService.messages$;
  
  constructor(private notificationService: NotificationService) {}
}

// Another component sending messages
@Component({
  selector: 'app-user-form',
  template: `
    <button (click)="save()">Save</button>
  `
})
export class UserFormComponent {
  constructor(private notificationService: NotificationService) {}
  
  save() {
    // ... save logic
    this.notificationService.showInfo('User saved successfully');
  }
}
```

### Subject Lifecycle

```typescript
import { Subject } from 'rxjs';

const subject$ = new Subject<number>();

// Subscribe
const sub1 = subject$.subscribe(val => console.log('Sub 1:', val));

// Emit values
subject$.next(1);
subject$.next(2);

// Add another subscriber
const sub2 = subject$.subscribe(val => console.log('Sub 2:', val));

subject$.next(3); // Both receive

// Complete the subject
subject$.complete();

subject$.next(4); // Neither receives (completed)

// New subscriptions immediately complete
const sub3 = subject$.subscribe({
  next: val => console.log('Sub 3:', val),
  complete: () => console.log('Sub 3 completed immediately')
});
```

## BehaviorSubject

BehaviorSubject stores the current value and emits it to new subscribers:

```typescript
import { BehaviorSubject } from 'rxjs';

// Requires initial value
const behaviorSubject$ = new BehaviorSubject<number>(0);

// First subscriber gets initial value immediately
behaviorSubject$.subscribe(val => console.log('Sub A:', val));
// Output: Sub A: 0

// Emit new value
behaviorSubject$.next(1);
// Output: Sub A: 1

// New subscriber gets current value immediately
behaviorSubject$.subscribe(val => console.log('Sub B:', val));
// Output: Sub B: 1

// Get current value synchronously
console.log('Current:', behaviorSubject$.value);
// Output: Current: 1
```

### BehaviorSubject for State Management

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({
  providedIn: 'root'
})
export class UserStateService {
  // Private subject with initial state
  private userSubject = new BehaviorSubject<User | null>(null);
  
  // Public observable
  user$: Observable<User | null> = this.userSubject.asObservable();
  
  // Synchronous access to current state
  get currentUser(): User | null {
    return this.userSubject.value;
  }
  
  // Update user
  setUser(user: User) {
    this.userSubject.next(user);
  }
  
  // Update specific properties
  updateUser(updates: Partial<User>) {
    const currentUser = this.currentUser;
    if (currentUser) {
      this.userSubject.next({ ...currentUser, ...updates });
    }
  }
  
  // Clear user
  clearUser() {
    this.userSubject.next(null);
  }
}

// Component
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="user$ | async as user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      <button (click)="updateName()">Update Name</button>
    </div>
  `
})
export class UserProfileComponent {
  user$ = this.userState.user$;
  
  constructor(private userState: UserStateService) {}
  
  updateName() {
    this.userState.updateUser({ name: 'Updated Name' });
  }
}
```

### BehaviorSubject for Loading State

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { finalize } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class LoadingService {
  private loadingSubject = new BehaviorSubject<boolean>(false);
  loading$: Observable<boolean> = this.loadingSubject.asObservable();
  
  show() {
    this.loadingSubject.next(true);
  }
  
  hide() {
    this.loadingSubject.next(false);
  }
  
  // Wrap an observable with loading state
  wrapWithLoading<T>(observable: Observable<T>): Observable<T> {
    this.show();
    return observable.pipe(
      finalize(() => this.hide())
    );
  }
}

// Component
@Component({
  selector: 'app-data-loader',
  template: `
    <div *ngIf="loadingService.loading$ | async">
      Loading...
    </div>
    <div *ngIf="data">
      {{ data | json }}
    </div>
    <button (click)="loadData()">Load Data</button>
  `
})
export class DataLoaderComponent {
  data: any = null;
  
  constructor(
    private http: HttpClient,
    public loadingService: LoadingService
  ) {}
  
  loadData() {
    const request$ = this.http.get('/api/data');
    
    this.loadingService
      .wrapWithLoading(request$)
      .subscribe(data => {
        this.data = data;
      });
  }
}
```

### BehaviorSubject for Form State

```typescript
import { Component } from '@angular/core';
import { BehaviorSubject, combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';

interface FormState {
  name: string;
  email: string;
  age: number;
}

@Component({
  selector: 'app-reactive-form',
  template: `
    <div>
      <input 
        [value]="name$ | async" 
        (input)="updateName($event)">
      
      <input 
        [value]="email$ | async" 
        (input)="updateEmail($event)">
      
      <input 
        type="number"
        [value]="age$ | async" 
        (input)="updateAge($event)">
      
      <div>Valid: {{ isValid$ | async }}</div>
      <div>Form State: {{ formState$ | async | json }}</div>
    </div>
  `
})
export class ReactiveFormComponent {
  private nameSubject = new BehaviorSubject<string>('');
  private emailSubject = new BehaviorSubject<string>('');
  private ageSubject = new BehaviorSubject<number>(0);
  
  name$ = this.nameSubject.asObservable();
  email$ = this.emailSubject.asObservable();
  age$ = this.ageSubject.asObservable();
  
  // Combine all form fields
  formState$ = combineLatest([
    this.name$,
    this.email$,
    this.age$
  ]).pipe(
    map(([name, email, age]) => ({ name, email, age }))
  );
  
  // Form validation
  isValid$ = this.formState$.pipe(
    map(state => 
      state.name.length > 0 && 
      state.email.includes('@') && 
      state.age >= 18
    )
  );
  
  updateName(event: any) {
    this.nameSubject.next(event.target.value);
  }
  
  updateEmail(event: any) {
    this.emailSubject.next(event.target.value);
  }
  
  updateAge(event: any) {
    this.ageSubject.next(parseInt(event.target.value, 10));
  }
}
```

## ReplaySubject

ReplaySubject records and replays n values to new subscribers:

```typescript
import { ReplaySubject } from 'rxjs';

// Replay last 2 values
const replaySubject$ = new ReplaySubject<number>(2);

replaySubject$.next(1);
replaySubject$.next(2);
replaySubject$.next(3);

// New subscriber gets last 2 values
replaySubject$.subscribe(val => console.log('Sub A:', val));
// Output:
// Sub A: 2
// Sub A: 3

replaySubject$.next(4);
// Output: Sub A: 4

// Another subscriber gets last 2 values
replaySubject$.subscribe(val => console.log('Sub B:', val));
// Output:
// Sub B: 3
// Sub B: 4
```

### ReplaySubject with Time Window

```typescript
import { ReplaySubject } from 'rxjs';

// Replay all values from last 500ms
const timedReplay$ = new ReplaySubject<number>(
  100,  // Buffer size
  500   // Time window in ms
);

timedReplay$.next(1);

setTimeout(() => {
  timedReplay$.next(2);
}, 300);

setTimeout(() => {
  timedReplay$.next(3);
}, 600);

setTimeout(() => {
  // Gets values 2 and 3 (within 500ms window)
  // Value 1 is too old
  timedReplay$.subscribe(val => console.log('Sub:', val));
}, 700);
```

### ReplaySubject for Caching

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ReplaySubject, Observable } from 'rxjs';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Injectable({
  providedIn: 'root'
})
export class ProductCacheService {
  private productsSubject = new ReplaySubject<Product[]>(1);
  products$ = this.productsSubject.asObservable();
  
  private loaded = false;
  
  constructor(private http: HttpClient) {}
  
  loadProducts() {
    if (!this.loaded) {
      this.http.get<Product[]>('/api/products').subscribe(
        products => {
          this.productsSubject.next(products);
          this.loaded = true;
        }
      );
    }
  }
  
  // New subscribers get cached data immediately
  getProducts(): Observable<Product[]> {
    this.loadProducts();
    return this.products$;
  }
}

// Component 1
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products$ | async">
      {{ product.name }}
    </div>
  `
})
export class ProductListComponent implements OnInit {
  products$ = this.productCache.getProducts();
  
  constructor(private productCache: ProductCacheService) {}
  
  ngOnInit() {
    // Gets data from cache or fetches
  }
}

// Component 2
@Component({
  selector: 'app-product-count',
  template: `
    <div>Total Products: {{ (products$ | async)?.length }}</div>
  `
})
export class ProductCountComponent implements OnInit {
  products$ = this.productCache.getProducts();
  
  constructor(private productCache: ProductCacheService) {}
  
  ngOnInit() {
    // Gets cached data immediately, no duplicate HTTP request
  }
}
```

### ReplaySubject for History

```typescript
import { Injectable } from '@angular/core';
import { ReplaySubject } from 'rxjs';

interface Action {
  type: string;
  timestamp: Date;
  data: any;
}

@Injectable({
  providedIn: 'root'
})
export class ActionHistoryService {
  // Keep last 50 actions
  private historySubject = new ReplaySubject<Action>(50);
  history$ = this.historySubject.asObservable();
  
  recordAction(type: string, data: any) {
    this.historySubject.next({
      type,
      timestamp: new Date(),
      data
    });
  }
}

// Component
@Component({
  selector: 'app-action-history',
  template: `
    <h3>Action History</h3>
    <div *ngFor="let action of history$ | async">
      {{ action.timestamp | date:'short' }} - 
      {{ action.type }}: 
      {{ action.data | json }}
    </div>
  `
})
export class ActionHistoryComponent {
  history$ = this.historyService.history$;
  
  constructor(private historyService: ActionHistoryService) {}
  
  // New subscriber immediately gets last 50 actions
}
```

## AsyncSubject

AsyncSubject only emits the last value when it completes:

```typescript
import { AsyncSubject } from 'rxjs';

const asyncSubject$ = new AsyncSubject<number>();

asyncSubject$.subscribe(val => console.log('Sub A:', val));

asyncSubject$.next(1);
asyncSubject$.next(2);
asyncSubject$.next(3);

// Nothing logged yet

asyncSubject$.subscribe(val => console.log('Sub B:', val));

// Complete the subject
asyncSubject$.complete();

// Now both subscribers get the last value
// Output:
// Sub A: 3
// Sub B: 3

// New subscribers also get the last value
asyncSubject$.subscribe(val => console.log('Sub C:', val));
// Output: Sub C: 3
```

### AsyncSubject for Final Result

```typescript
import { Injectable } from '@angular/core';
import { AsyncSubject, Observable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class ComputationService {
  computeAsync(data: number[]): Observable<number> {
    const subject = new AsyncSubject<number>();
    
    // Simulate long computation
    setTimeout(() => {
      let result = 0;
      for (let i = 0; i < data.length; i++) {
        result += data[i];
        // Intermediate results not emitted
        subject.next(result);
      }
      
      // Only final result is emitted
      subject.complete();
    }, 2000);
    
    return subject.asObservable();
  }
}

// Component
@Component({
  selector: 'app-computation',
  template: `
    <div>Result: {{ result$ | async }}</div>
  `
})
export class ComputationComponent {
  result$ = this.computationService.computeAsync([1, 2, 3, 4, 5]);
  
  constructor(private computationService: ComputationService) {}
  
  // Only displays final result (15)
}
```

## Subject vs Observable

```typescript
import { Observable, Subject } from 'rxjs';

// Observable - unicast, lazy
const observable$ = new Observable(subscriber => {
  console.log('Observable execution');
  subscriber.next(Math.random());
});

observable$.subscribe(val => console.log('Obs Sub 1:', val));
observable$.subscribe(val => console.log('Obs Sub 2:', val));
// Two executions, different random numbers

// Subject - multicast, eager
const subject$ = new Subject<number>();

subject$.subscribe(val => console.log('Sub Sub 1:', val));
subject$.subscribe(val => console.log('Sub Sub 2:', val));

subject$.next(Math.random());
// One execution, same random number for both
```

### Converting Cold to Hot

```typescript
import { Observable, Subject } from 'rxjs';
import { share, shareReplay } from 'rxjs/operators';

// Cold observable (HTTP request)
const cold$ = this.http.get('/api/data');

// Method 1: Manual with Subject
const subject = new Subject();
cold$.subscribe(subject);
const hot$ = subject.asObservable();

// Method 2: Using share operator
const hot2$ = cold$.pipe(share());

// Method 3: Using shareReplay
const hot3$ = cold$.pipe(shareReplay(1));

// All subscribers share the same execution
```

## Use Cases

### Event Bus

```typescript
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';
import { filter, map } from 'rxjs/operators';

interface Event {
  type: string;
  payload?: any;
}

@Injectable({
  providedIn: 'root'
})
export class EventBusService {
  private eventSubject = new Subject<Event>();
  
  // Emit event
  emit(type: string, payload?: any) {
    this.eventSubject.next({ type, payload });
  }
  
  // Listen to specific event type
  on(eventType: string): Observable<any> {
    return this.eventSubject.pipe(
      filter(event => event.type === eventType),
      map(event => event.payload)
    );
  }
}

// Component A - emits events
@Component({
  selector: 'app-producer',
  template: `<button (click)="sendEvent()">Send Event</button>`
})
export class ProducerComponent {
  constructor(private eventBus: EventBusService) {}
  
  sendEvent() {
    this.eventBus.emit('USER_ACTION', { userId: 123 });
  }
}

// Component B - listens to events
@Component({
  selector: 'app-consumer',
  template: `<div>{{ lastEvent | json }}</div>`
})
export class ConsumerComponent implements OnInit {
  lastEvent: any = null;
  
  constructor(private eventBus: EventBusService) {}
  
  ngOnInit() {
    this.eventBus.on('USER_ACTION').subscribe(payload => {
      this.lastEvent = payload;
    });
  }
}
```

### Component Communication

```typescript
// Parent-Child communication via service
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

@Injectable()
export class DataSharingService {
  private dataSubject = new Subject<any>();
  data$ = this.dataSubject.asObservable();
  
  sendData(data: any) {
    this.dataSubject.next(data);
  }
}

// Parent component
@Component({
  selector: 'app-parent',
  template: `
    <button (click)="sendToChild()">Send to Child</button>
    <app-child></app-child>
  `,
  providers: [DataSharingService]
})
export class ParentComponent {
  constructor(private dataSharing: DataSharingService) {}
  
  sendToChild() {
    this.dataSharing.sendData({ message: 'Hello from parent' });
  }
}

// Child component
@Component({
  selector: 'app-child',
  template: `<div>{{ receivedData | json }}</div>`
})
export class ChildComponent implements OnInit {
  receivedData: any = null;
  
  constructor(private dataSharing: DataSharingService) {}
  
  ngOnInit() {
    this.dataSharing.data$.subscribe(data => {
      this.receivedData = data;
    });
  }
}
```

## Common Mistakes

### Mistake 1: Not Completing Subjects

```typescript
// BAD: Subject never completes
@Injectable({
  providedIn: 'root'
})
export class BadService {
  private subject = new Subject<any>();
  data$ = this.subject.asObservable();
  
  // Subject lives forever, potential memory leak
}

// GOOD: Complete in ngOnDestroy
@Injectable()
export class GoodService implements OnDestroy {
  private subject = new Subject<any>();
  data$ = this.subject.asObservable();
  
  ngOnDestroy() {
    this.subject.complete();
  }
}
```

### Mistake 2: Exposing Subject Directly

```typescript
// BAD: Subject is public
export class BadService {
  public dataSubject = new Subject<any>();
  
  // Anyone can call dataSubject.next() or complete()
}

// GOOD: Expose as Observable
export class GoodService {
  private dataSubject = new Subject<any>();
  data$ = this.dataSubject.asObservable();
  
  // Only service can call next()
  sendData(data: any) {
    this.dataSubject.next(data);
  }
}
```

### Mistake 3: Using Subject Instead of BehaviorSubject

```typescript
// BAD: New subscribers miss current state
export class BadStateService {
  private stateSubject = new Subject<any>();
  state$ = this.stateSubject.asObservable();
  
  setState(state: any) {
    this.stateSubject.next(state);
  }
}
// Late subscribers don't get current state!

// GOOD: BehaviorSubject for state
export class GoodStateService {
  private stateSubject = new BehaviorSubject<any>(null);
  state$ = this.stateSubject.asObservable();
  
  setState(state: any) {
    this.stateSubject.next(state);
  }
}
// Late subscribers immediately get current state
```

## Best Practices

### 1. Use Appropriate Subject Type

```typescript
// Use Subject for events
private eventSubject = new Subject<Event>();

// Use BehaviorSubject for state
private stateSubject = new BehaviorSubject<State>(initialState);

// Use ReplaySubject for caching
private cacheSubject = new ReplaySubject<Data>(1);

// Use AsyncSubject for final results
private resultSubject = new AsyncSubject<Result>();
```

### 2. Always Complete Subjects

```typescript
@Injectable()
export class ProperService implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Encapsulate Subjects

```typescript
@Injectable({
  providedIn: 'root'
})
export class EncapsulatedService {
  // Private subject
  private dataSubject = new BehaviorSubject<any>(null);
  
  // Public read-only observable
  readonly data$ = this.dataSubject.asObservable();
  
  // Public methods to modify
  updateData(data: any) {
    this.dataSubject.next(data);
  }
}
```

## Interview Questions

### Q1: What's the difference between Subject and Observable?

**Answer:** Subject is both Observable and Observer - it can emit values (like an Observer) and be subscribed to (like an Observable). It multicasts to multiple subscribers, meaning all subscribers receive the same values. Regular Observables are unicast - each subscriber gets independent execution.

### Q2: When should you use BehaviorSubject vs Subject?

**Answer:** Use BehaviorSubject when you need to represent state that always has a current value. New subscribers immediately receive the current value. Use plain Subject for events that don't have a "current" concept and where missing past events is acceptable.

### Q3: What's the difference between ReplaySubject and BehaviorSubject?

**Answer:** BehaviorSubject stores and replays only the last emitted value. ReplaySubject can store and replay multiple values (configurable buffer size) or all values within a time window. BehaviorSubject requires an initial value, ReplaySubject doesn't.

### Q4: Why should you not expose Subjects directly?

**Answer:** Exposing Subjects directly allows external code to call next(), error(), or complete() on them, violating encapsulation. Instead, expose the Subject as an Observable using asObservable(), and provide controlled methods to emit values.

### Q5: What is AsyncSubject used for?

**Answer:** AsyncSubject only emits the last value when it completes. It's useful for computations where you only care about the final result, or for converting Promises to Observables where you only want the resolved value.

## Key Takeaways

1. Subjects are both Observable and Observer
2. Subjects multicast values to all subscribers
3. BehaviorSubject stores current value, requires initial value
4. ReplaySubject buffers n values and replays to new subscribers
5. AsyncSubject only emits last value when completed
6. Use Subject for events, BehaviorSubject for state
7. Always expose Subjects as Observable using asObservable()
8. Complete Subjects in cleanup to prevent memory leaks
9. Choose the right Subject type for your use case
10. Subjects convert cold observables to hot

## Resources

### Official Documentation

- [RxJS Subject](https://rxjs.dev/guide/subject)
- [BehaviorSubject API](https://rxjs.dev/api/index/class/BehaviorSubject)
- [ReplaySubject API](https://rxjs.dev/api/index/class/ReplaySubject)

### Articles

- [Understanding Subjects](https://angular.io/guide/rx-library#subjects)
- [When to use which Subject](https://blog.angular-university.io/how-to-build-angular2-apps-using-rxjs-observable-data-services-pitfalls-to-avoid/)
- [Subject Best Practices](https://angular.dev/guide/practical-observable-usage)

### Videos

- "RxJS Subjects Explained"
- "State Management with BehaviorSubject"
