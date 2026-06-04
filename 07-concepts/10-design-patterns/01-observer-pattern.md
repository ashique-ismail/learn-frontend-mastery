# Observer Pattern

## Overview

The Observer pattern is a behavioral design pattern that defines a one-to-many dependency between objects. When one object (the subject) changes state, all its dependents (observers) are automatically notified and updated. This pattern is fundamental to event-driven programming and reactive systems.

## Core Concepts

### Subject-Observer Relationship

```
┌─────────────────┐
│    Subject      │
├─────────────────┤
│ - observers[]   │
│ + attach()      │
│ + detach()      │
│ + notify()      │
└────────┬────────┘
         │
         │ notifies
         ▼
┌─────────────────┐
│    Observer     │
├─────────────────┤
│ + update()      │
└─────────────────┘
         △
         │
    ┌────┴────┐
    │         │
┌───┴───┐ ┌──┴────┐
│Obs A  │ │Obs B  │
└───────┘ └───────┘
```

### Key Components

1. **Subject**: Maintains list of observers, provides methods to attach/detach
2. **Observer**: Defines update interface for receiving notifications
3. **ConcreteSubject**: Stores state, sends notifications when state changes
4. **ConcreteObserver**: Implements update to keep state consistent with subject

## Implementation Patterns

### Basic Observer Pattern (Vanilla JavaScript)

```javascript
// Subject interface
class Subject {
  constructor() {
    this.observers = [];
  }

  attach(observer) {
    if (!this.observers.includes(observer)) {
      this.observers.push(observer);
      console.log('Observer attached');
    }
  }

  detach(observer) {
    const index = this.observers.indexOf(observer);
    if (index > -1) {
      this.observers.splice(index, 1);
      console.log('Observer detached');
    }
  }

  notify(data) {
    console.log(`Notifying ${this.observers.length} observers`);
    this.observers.forEach(observer => observer.update(data));
  }
}

// Observer interface
class Observer {
  update(data) {
    throw new Error('Observer.update() must be implemented');
  }
}

// Concrete Subject
class StockTicker extends Subject {
  constructor() {
    super();
    this.stockData = {};
  }

  setStockPrice(symbol, price) {
    this.stockData[symbol] = price;
    this.notify({ symbol, price });
  }

  getStockPrice(symbol) {
    return this.stockData[symbol];
  }
}

// Concrete Observers
class StockDisplay extends Observer {
  constructor(name) {
    super();
    this.name = name;
  }

  update(data) {
    console.log(`${this.name} - Stock Update: ${data.symbol} = $${data.price}`);
  }
}

class StockAlert extends Observer {
  constructor(symbol, threshold) {
    super();
    this.symbol = symbol;
    this.threshold = threshold;
  }

  update(data) {
    if (data.symbol === this.symbol && data.price > this.threshold) {
      console.log(`ALERT: ${data.symbol} exceeded threshold! Price: $${data.price}`);
    }
  }
}

// Usage
const ticker = new StockTicker();
const display1 = new StockDisplay('Display-1');
const display2 = new StockDisplay('Display-2');
const alert = new StockAlert('AAPL', 150);

ticker.attach(display1);
ticker.attach(display2);
ticker.attach(alert);

ticker.setStockPrice('AAPL', 145); // All observers notified
ticker.setStockPrice('AAPL', 155); // Alert triggered

ticker.detach(display2);
ticker.setStockPrice('GOOGL', 2800); // Only display1 and alert notified
```

### React Observer Pattern with Custom Hook

```typescript
// useObservable.ts
import { useEffect, useState } from 'react';

interface Observer<T> {
  update: (data: T) => void;
}

class Observable<T> {
  private observers: Set<Observer<T>> = new Set();

  subscribe(observer: Observer<T>): () => void {
    this.observers.add(observer);
    return () => this.observers.delete(observer);
  }

  notify(data: T): void {
    this.observers.forEach(observer => observer.update(data));
  }

  getObserverCount(): number {
    return this.observers.size;
  }
}

// Custom hook for subscribing to observables
function useObservable<T>(observable: Observable<T>, initialValue: T): T {
  const [value, setValue] = useState<T>(initialValue);

  useEffect(() => {
    const observer: Observer<T> = {
      update: (data: T) => setValue(data),
    };

    const unsubscribe = observable.subscribe(observer);
    return unsubscribe;
  }, [observable]);

  return value;
}

// Store implementation
interface User {
  id: string;
  name: string;
  email: string;
}

class UserStore extends Observable<User | null> {
  private user: User | null = null;

  setUser(user: User | null): void {
    this.user = user;
    this.notify(user);
  }

  getUser(): User | null {
    return this.user;
  }

  updateUserName(name: string): void {
    if (this.user) {
      this.user = { ...this.user, name };
      this.notify(this.user);
    }
  }
}

// Create singleton store
export const userStore = new UserStore();

// Component usage
import React from 'react';

export const UserProfile: React.FC = () => {
  const user = useObservable(userStore, null);

  if (!user) {
    return <div>No user logged in</div>;
  }

  return (
    <div className="user-profile">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => userStore.updateUserName('New Name')}>
        Update Name
      </button>
    </div>
  );
};

export const UserGreeting: React.FC = () => {
  const user = useObservable(userStore, null);

  return (
    <div className="greeting">
      {user ? `Welcome back, ${user.name}!` : 'Please log in'}
    </div>
  );
};

// Both components automatically re-render when user changes
```

### Angular Observable with RxJS

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, Subject } from 'rxjs';
import { filter, map, distinctUntilChanged } from 'rxjs/operators';

export interface User {
  id: string;
  name: string;
  email: string;
  role: string;
}

export interface Notification {
  id: string;
  message: string;
  type: 'info' | 'success' | 'warning' | 'error';
  timestamp: Date;
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  // BehaviorSubject stores current value and emits on subscription
  private userSubject = new BehaviorSubject<User | null>(null);
  public user$: Observable<User | null> = this.userSubject.asObservable();

  // Subject doesn't store value, only emits new events
  private notificationSubject = new Subject<Notification>();
  public notifications$: Observable<Notification> = this.notificationSubject.asObservable();

  // Derived observables
  public isLoggedIn$: Observable<boolean> = this.user$.pipe(
    map(user => user !== null),
    distinctUntilChanged()
  );

  public userName$: Observable<string> = this.user$.pipe(
    filter(user => user !== null),
    map(user => user!.name),
    distinctUntilChanged()
  );

  public isAdmin$: Observable<boolean> = this.user$.pipe(
    map(user => user?.role === 'admin'),
    distinctUntilChanged()
  );

  setUser(user: User | null): void {
    this.userSubject.next(user);
    
    if (user) {
      this.notificationSubject.next({
        id: Date.now().toString(),
        message: `Welcome back, ${user.name}!`,
        type: 'success',
        timestamp: new Date()
      });
    }
  }

  updateUser(updates: Partial<User>): void {
    const currentUser = this.userSubject.value;
    if (currentUser) {
      const updatedUser = { ...currentUser, ...updates };
      this.userSubject.next(updatedUser);
      
      this.notificationSubject.next({
        id: Date.now().toString(),
        message: 'Profile updated successfully',
        type: 'success',
        timestamp: new Date()
      });
    }
  }

  logout(): void {
    this.userSubject.next(null);
    this.notificationSubject.next({
      id: Date.now().toString(),
      message: 'Logged out successfully',
      type: 'info',
      timestamp: new Date()
    });
  }

  getCurrentUser(): User | null {
    return this.userSubject.value;
  }
}

// user-profile.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject, takeUntil } from 'rxjs';
import { UserService, User } from './user.service';

@Component({
  selector: 'app-user-profile',
  template: `
    <div class="user-profile" *ngIf="user$ | async as user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      <p>Role: {{ user.role }}</p>
      
      <div *ngIf="isAdmin$ | async" class="admin-badge">
        Admin
      </div>

      <button (click)="updateName()">Update Name</button>
      <button (click)="logout()">Logout</button>
    </div>

    <div class="logged-out" *ngIf="!(isLoggedIn$ | async)">
      <p>Please log in</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit, OnDestroy {
  user$ = this.userService.user$;
  isLoggedIn$ = this.userService.isLoggedIn$;
  isAdmin$ = this.userService.isAdmin$;

  private destroy$ = new Subject<void>();

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    // Subscribe with automatic cleanup
    this.userService.user$
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => {
        console.log('User changed:', user);
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  updateName(): void {
    this.userService.updateUser({ name: 'Updated Name' });
  }

  logout(): void {
    this.userService.logout();
  }
}

// notification.component.ts
import { Component } from '@angular/core';
import { UserService, Notification } from './user.service';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-notifications',
  template: `
    <div class="notifications">
      <div *ngFor="let notification of notifications$ | async"
           [class]="'notification ' + notification.type">
        <span>{{ notification.message }}</span>
        <small>{{ notification.timestamp | date:'short' }}</small>
      </div>
    </div>
  `
})
export class NotificationComponent {
  notifications$: Observable<Notification>;

  constructor(private userService: UserService) {
    this.notifications$ = this.userService.notifications$;
  }
}
```

### Event Emitter Pattern

```typescript
// event-emitter.ts
type EventHandler<T = any> = (data: T) => void;

class EventEmitter {
  private events: Map<string, Set<EventHandler>> = new Map();

  on(event: string, handler: EventHandler): () => void {
    if (!this.events.has(event)) {
      this.events.set(event, new Set());
    }
    
    this.events.get(event)!.add(handler);
    
    // Return unsubscribe function
    return () => this.off(event, handler);
  }

  once(event: string, handler: EventHandler): void {
    const wrappedHandler = (data: any) => {
      handler(data);
      this.off(event, wrappedHandler);
    };
    
    this.on(event, wrappedHandler);
  }

  off(event: string, handler: EventHandler): void {
    const handlers = this.events.get(event);
    if (handlers) {
      handlers.delete(handler);
      if (handlers.size === 0) {
        this.events.delete(event);
      }
    }
  }

  emit(event: string, data?: any): void {
    const handlers = this.events.get(event);
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler(data);
        } catch (error) {
          console.error(`Error in event handler for "${event}":`, error);
        }
      });
    }
  }

  listenerCount(event: string): number {
    return this.events.get(event)?.size || 0;
  }

  removeAllListeners(event?: string): void {
    if (event) {
      this.events.delete(event);
    } else {
      this.events.clear();
    }
  }
}

// Application events
interface AppEvents {
  'user:login': { userId: string; timestamp: Date };
  'user:logout': { userId: string };
  'cart:add': { productId: string; quantity: number };
  'cart:remove': { productId: string };
  'order:placed': { orderId: string; total: number };
}

class TypedEventEmitter<TEvents extends Record<string, any>> {
  private emitter = new EventEmitter();

  on<K extends keyof TEvents>(
    event: K,
    handler: (data: TEvents[K]) => void
  ): () => void {
    return this.emitter.on(event as string, handler);
  }

  once<K extends keyof TEvents>(
    event: K,
    handler: (data: TEvents[K]) => void
  ): void {
    this.emitter.once(event as string, handler);
  }

  off<K extends keyof TEvents>(
    event: K,
    handler: (data: TEvents[K]) => void
  ): void {
    this.emitter.off(event as string, handler);
  }

  emit<K extends keyof TEvents>(event: K, data: TEvents[K]): void {
    this.emitter.emit(event as string, data);
  }
}

// Usage
const appEvents = new TypedEventEmitter<AppEvents>();

// Subscribe to events
const unsubscribeLogin = appEvents.on('user:login', (data) => {
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
});

appEvents.on('cart:add', (data) => {
  console.log(`Added ${data.quantity} of product ${data.productId}`);
});

appEvents.once('order:placed', (data) => {
  console.log(`Order ${data.orderId} placed! Total: $${data.total}`);
  // Send confirmation email
});

// Emit events
appEvents.emit('user:login', {
  userId: 'user123',
  timestamp: new Date()
});

appEvents.emit('cart:add', {
  productId: 'prod456',
  quantity: 2
});

appEvents.emit('order:placed', {
  orderId: 'order789',
  total: 99.99
});

// Cleanup
unsubscribeLogin();
```

### Push vs Pull Observer

```typescript
// Push-based (observers receive data)
class PushObservable<T> {
  private observers: Set<(data: T) => void> = new Set();

  subscribe(observer: (data: T) => void): () => void {
    this.observers.add(observer);
    return () => this.observers.delete(observer);
  }

  next(data: T): void {
    this.observers.forEach(observer => observer(data));
  }
}

// Pull-based (observers query data)
class PullObservable<T> {
  private observers: Set<() => void> = new Set();
  private value: T;

  constructor(initialValue: T) {
    this.value = initialValue;
  }

  subscribe(observer: () => void): () => void {
    this.observers.add(observer);
    return () => this.observers.delete(observer);
  }

  getValue(): T {
    return this.value;
  }

  setValue(value: T): void {
    this.value = value;
    this.observers.forEach(observer => observer());
  }
}

// Push example - data pushed to observers
const pushTemp = new PushObservable<number>();

pushTemp.subscribe(temp => {
  console.log(`Temperature is now: ${temp}°C`);
});

pushTemp.next(22); // Temperature is now: 22°C
pushTemp.next(24); // Temperature is now: 24°C

// Pull example - observers pull data when notified
const pullTemp = new PullObservable<number>(20);

pullTemp.subscribe(() => {
  const temp = pullTemp.getValue();
  console.log(`Check temperature: ${temp}°C`);
});

pullTemp.setValue(22); // Check temperature: 22°C
pullTemp.setValue(24); // Check temperature: 24°C
```

### RxJS Subject Types

```typescript
import { 
  Subject, 
  BehaviorSubject, 
  ReplaySubject, 
  AsyncSubject 
} from 'rxjs';

// 1. Subject - no initial value, no replay
const subject = new Subject<number>();

subject.subscribe(value => console.log('Sub A:', value));
subject.next(1);
subject.next(2);
subject.subscribe(value => console.log('Sub B:', value)); // Misses 1 and 2
subject.next(3);
// Output:
// Sub A: 1
// Sub A: 2
// Sub A: 3
// Sub B: 3

// 2. BehaviorSubject - has initial value, replays last value
const behaviorSubject = new BehaviorSubject<number>(0);

behaviorSubject.subscribe(value => console.log('Sub A:', value)); // Gets 0
behaviorSubject.next(1);
behaviorSubject.next(2);
behaviorSubject.subscribe(value => console.log('Sub B:', value)); // Gets 2
behaviorSubject.next(3);
// Output:
// Sub A: 0
// Sub A: 1
// Sub A: 2
// Sub B: 2
// Sub A: 3
// Sub B: 3

// 3. ReplaySubject - replays N values
const replaySubject = new ReplaySubject<number>(2); // Buffer size 2

replaySubject.next(1);
replaySubject.next(2);
replaySubject.next(3);
replaySubject.subscribe(value => console.log('Sub A:', value)); // Gets 2, 3
replaySubject.next(4);
// Output:
// Sub A: 2
// Sub A: 3
// Sub A: 4

// 4. AsyncSubject - emits only last value on completion
const asyncSubject = new AsyncSubject<number>();

asyncSubject.subscribe(value => console.log('Sub A:', value));
asyncSubject.next(1);
asyncSubject.next(2);
asyncSubject.next(3);
asyncSubject.subscribe(value => console.log('Sub B:', value));
asyncSubject.complete(); // Only now do subscribers receive value 3
// Output:
// Sub A: 3
// Sub B: 3

// Practical example - Theme service
class ThemeService {
  // Use BehaviorSubject for current theme
  private themeSubject = new BehaviorSubject<'light' | 'dark'>('light');
  public theme$ = this.themeSubject.asObservable();

  // Use ReplaySubject for theme history
  private historySubject = new ReplaySubject<string>(5); // Last 5 changes
  public history$ = this.historySubject.asObservable();

  setTheme(theme: 'light' | 'dark'): void {
    this.themeSubject.next(theme);
    this.historySubject.next(`Theme changed to ${theme} at ${new Date().toISOString()}`);
  }

  getCurrentTheme(): 'light' | 'dark' {
    return this.themeSubject.value;
  }
}
```

### Memory Leak Prevention

```typescript
// React - Cleanup subscriptions
import { useEffect, useRef } from 'react';

function useEventListener<T>(
  observable: Observable<T>,
  handler: (data: T) => void
) {
  const handlerRef = useRef(handler);

  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);

  useEffect(() => {
    const observer = {
      update: (data: T) => handlerRef.current(data)
    };

    const unsubscribe = observable.subscribe(observer);

    return () => {
      unsubscribe(); // Cleanup on unmount
    };
  }, [observable]);
}

// Angular - takeUntil pattern
import { Component, OnDestroy } from '@angular/core';
import { Subject, takeUntil } from 'rxjs';

@Component({
  selector: 'app-example',
  template: '<div>{{ data }}</div>'
})
export class ExampleComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  data: any;

  constructor(private dataService: DataService) {
    // All subscriptions automatically unsubscribe on destroy
    this.dataService.data$
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => this.data = data);

    this.dataService.updates$
      .pipe(takeUntil(this.destroy$))
      .subscribe(update => console.log(update));
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Weak reference observers (advanced)
class WeakObservable<T> {
  private observers = new Set<WeakRef<Observer<T>>>();
  private registry = new FinalizationRegistry((cleanup: () => void) => {
    cleanup();
  });

  subscribe(observer: Observer<T>): () => void {
    const weakRef = new WeakRef(observer);
    this.observers.add(weakRef);

    const cleanup = () => this.observers.delete(weakRef);
    this.registry.register(observer, cleanup);

    return cleanup;
  }

  notify(data: T): void {
    for (const weakRef of this.observers) {
      const observer = weakRef.deref();
      if (observer) {
        observer.update(data);
      } else {
        this.observers.delete(weakRef);
      }
    }
  }
}
```

### Async Observer Pattern

```typescript
// Async observers
interface AsyncObserver<T> {
  update(data: T): Promise<void>;
}

class AsyncObservable<T> {
  private observers: Set<AsyncObserver<T>> = new Set();

  subscribe(observer: AsyncObserver<T>): () => void {
    this.observers.add(observer);
    return () => this.observers.delete(observer);
  }

  async notify(data: T): Promise<void> {
    const promises = Array.from(this.observers).map(observer =>
      observer.update(data).catch(error => {
        console.error('Observer error:', error);
      })
    );

    await Promise.all(promises);
  }

  async notifySequential(data: T): Promise<void> {
    for (const observer of this.observers) {
      try {
        await observer.update(data);
      } catch (error) {
        console.error('Observer error:', error);
      }
    }
  }
}

// Usage example
class EmailNotifier implements AsyncObserver<Order> {
  async update(order: Order): Promise<void> {
    console.log('Sending email...');
    await this.sendEmail(order);
    console.log('Email sent');
  }

  private async sendEmail(order: Order): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, 1000));
  }
}

class InventoryUpdater implements AsyncObserver<Order> {
  async update(order: Order): Promise<void> {
    console.log('Updating inventory...');
    await this.updateInventory(order);
    console.log('Inventory updated');
  }

  private async updateInventory(order: Order): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, 500));
  }
}

interface Order {
  id: string;
  items: string[];
}

const orderProcessor = new AsyncObservable<Order>();
orderProcessor.subscribe(new EmailNotifier());
orderProcessor.subscribe(new InventoryUpdater());

// Parallel execution
await orderProcessor.notify({
  id: 'order123',
  items: ['item1', 'item2']
});

// Sequential execution
await orderProcessor.notifySequential({
  id: 'order456',
  items: ['item3']
});
```

### Advanced: Priority Observers

```typescript
interface PriorityObserver<T> {
  priority: number;
  update(data: T): void;
}

class PriorityObservable<T> {
  private observers: PriorityObserver<T>[] = [];

  subscribe(observer: PriorityObserver<T>): () => void {
    this.observers.push(observer);
    this.observers.sort((a, b) => b.priority - a.priority);

    return () => {
      const index = this.observers.indexOf(observer);
      if (index > -1) {
        this.observers.splice(index, 1);
      }
    };
  }

  notify(data: T): void {
    // Notify in priority order (highest first)
    for (const observer of this.observers) {
      observer.update(data);
    }
  }
}

// Usage
class CriticalLogger implements PriorityObserver<string> {
  priority = 10;
  
  update(data: string): void {
    console.log(`[CRITICAL] ${data}`);
  }
}

class StandardLogger implements PriorityObserver<string> {
  priority = 5;
  
  update(data: string): void {
    console.log(`[INFO] ${data}`);
  }
}

class DebugLogger implements PriorityObserver<string> {
  priority = 1;
  
  update(data: string): void {
    console.log(`[DEBUG] ${data}`);
  }
}

const logger = new PriorityObservable<string>();
logger.subscribe(new DebugLogger());
logger.subscribe(new CriticalLogger());
logger.subscribe(new StandardLogger());

logger.notify('System event'); // Logs in priority order
```

## Common Mistakes

### 1. Memory Leaks from Forgotten Unsubscribe

```typescript
// BAD - Memory leak
class BadComponent {
  constructor(private dataService: DataService) {
    this.dataService.data$.subscribe(data => {
      this.handleData(data);
    }); // Never unsubscribed!
  }
}

// GOOD - Proper cleanup
class GoodComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  constructor(private dataService: DataService) {
    this.dataService.data$
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => this.handleData(data));
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. Circular Dependencies

```typescript
// BAD - Circular observer dependency
class ObserverA extends Observable {
  constructor(private observerB: ObserverB) {
    super();
  }

  notify() {
    super.notify();
    this.observerB.update(); // Can cause infinite loop
  }
}

// GOOD - Use mediator pattern
class Mediator {
  private observerA: ObserverA;
  private observerB: ObserverB;

  notifyA(): void {
    this.observerA.update();
  }

  notifyB(): void {
    this.observerB.update();
  }
}
```

### 3. Synchronous Notifications Causing Stack Overflow

```typescript
// BAD - Synchronous cascade
class BadObservable {
  notify(data: any): void {
    this.observers.forEach(observer => {
      observer.update(data); // Can trigger more notifications
    });
  }
}

// GOOD - Async or batched notifications
class GoodObservable {
  private pendingNotifications = new Set<any>();
  private isNotifying = false;

  notify(data: any): void {
    this.pendingNotifications.add(data);
    
    if (!this.isNotifying) {
      queueMicrotask(() => this.processNotifications());
    }
  }

  private processNotifications(): void {
    this.isNotifying = true;
    
    const notifications = Array.from(this.pendingNotifications);
    this.pendingNotifications.clear();
    
    notifications.forEach(data => {
      this.observers.forEach(observer => observer.update(data));
    });
    
    this.isNotifying = false;
  }
}
```

### 4. Not Handling Observer Errors

```typescript
// BAD - One error stops all notifications
notify(data: any): void {
  this.observers.forEach(observer => observer.update(data));
}

// GOOD - Isolated error handling
notify(data: any): void {
  this.observers.forEach(observer => {
    try {
      observer.update(data);
    } catch (error) {
      console.error('Observer error:', error);
      // Optionally remove failing observer
      // this.observers.delete(observer);
    }
  });
}
```

## Best Practices

1. **Always clean up subscriptions** to prevent memory leaks
2. **Use typed events** for better type safety
3. **Handle observer errors** gracefully
4. **Avoid circular dependencies** between observers
5. **Use appropriate Subject types** in RxJS (BehaviorSubject vs Subject)
6. **Consider async notifications** for long-running observers
7. **Document event contracts** clearly
8. **Use weak references** for optional observers
9. **Batch notifications** when appropriate
10. **Test observer interactions** thoroughly

## When to Use Observer Pattern

### Good Use Cases

- Event handling systems
- Real-time data updates (stock tickers, chat, notifications)
- State management (Redux, MobX, Angular services)
- Model-View synchronization (MVC, MVVM)
- Pub/sub messaging systems
- UI component updates
- Logging and monitoring systems

### When to Avoid

- Simple one-to-one relationships (use direct method calls)
- Tight coupling is acceptable (use direct references)
- Performance-critical paths (observer notifications have overhead)
- Complex observer dependencies (consider mediator pattern)

## Interview Questions

### Q1: What's the difference between Observer pattern and Pub/Sub?

**Answer:** While similar, they have key differences:

- **Observer**: Subject knows its observers, tight coupling
- **Pub/Sub**: Publishers and subscribers are decoupled through an event channel/broker

```typescript
// Observer - tight coupling
class Subject {
  private observers: Observer[] = [];
  attach(observer: Observer) { this.observers.push(observer); }
}

// Pub/Sub - loose coupling
class EventBus {
  private channels = new Map<string, Function[]>();
  
  publish(channel: string, data: any) {
    this.channels.get(channel)?.forEach(handler => handler(data));
  }
  
  subscribe(channel: string, handler: Function) {
    if (!this.channels.has(channel)) {
      this.channels.set(channel, []);
    }
    this.channels.get(channel)!.push(handler);
  }
}
```

### Q2: How do you prevent memory leaks with observers in React?

**Answer:** Use cleanup functions in useEffect:

```typescript
useEffect(() => {
  const unsubscribe = observable.subscribe(handleData);
  return () => unsubscribe(); // Cleanup on unmount
}, [observable]);
```

### Q3: What's the difference between Subject and BehaviorSubject in RxJS?

**Answer:**
- **Subject**: No initial value, new subscribers don't receive previous values
- **BehaviorSubject**: Has initial value, new subscribers immediately receive the last emitted value

### Q4: How would you implement an observer pattern with priority?

**Answer:** Maintain sorted observer list and notify in priority order (see Priority Observers example above).

### Q5: How do you handle async observers that might fail?

**Answer:** Use Promise.allSettled() or individual try-catch blocks to isolate failures:

```typescript
async notify(data: T): Promise<void> {
  const promises = Array.from(this.observers).map(observer =>
    observer.update(data).catch(error => ({ error }))
  );
  const results = await Promise.allSettled(promises);
  // Handle failures appropriately
}
```

### Q6: What's the push vs pull model in Observer pattern?

**Answer:**
- **Push**: Subject sends data to observers (observers are passive)
- **Pull**: Subject notifies observers, they query the data (observers are active)

### Q7: How do you avoid circular dependencies with observers?

**Answer:** Use a mediator/event bus, add re-entrance guards, or make notifications async.

### Q8: When should you use ReplaySubject over BehaviorSubject?

**Answer:** Use ReplaySubject when you need to replay multiple values (e.g., event history, audit logs) vs BehaviorSubject which only replays the last value (e.g., current state).

## Key Takeaways

1. Observer pattern enables one-to-many dependency between objects
2. Loose coupling allows subjects and observers to vary independently
3. RxJS provides powerful observable implementations (Subject, BehaviorSubject, ReplaySubject)
4. Always clean up subscriptions to prevent memory leaks
5. Handle observer errors gracefully to prevent cascade failures
6. Push model sends data, pull model notifies and observers query
7. TypeScript enables type-safe event systems
8. Consider async notifications for long-running observers
9. Use appropriate Subject types for your use case
10. Test observer interactions and edge cases thoroughly

## Resources

- [RxJS Documentation](https://rxjs.dev/)
- [Gang of Four Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- [Refactoring Guru: Observer Pattern](https://refactoring.guru/design-patterns/observer)
- [Angular RxJS Best Practices](https://angular.io/guide/rx-library)
- [React State Management](https://react.dev/learn/managing-state)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
