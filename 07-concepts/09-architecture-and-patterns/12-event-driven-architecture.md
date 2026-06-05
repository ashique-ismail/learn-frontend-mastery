# Event-Driven Architecture

## The Idea

**In plain English:** Event-Driven Architecture is a way of building software where different parts of a program communicate by broadcasting announcements (called "events") and listening for announcements they care about, instead of directly calling each other. An event is just a signal that says "something happened" — like a button was clicked, a purchase was made, or a user logged in.

**Real-world analogy:** Think of a school announcement system. The principal (event producer) makes an announcement over the PA system (event bus). Every classroom teacher (event consumer) hears it, but only the gym teacher reacts to "fire drill announced" while the cafeteria staff reacts to "lunch period starting."

- The principal = the event producer (the part of the app that triggers something)
- The PA system = the event bus (the channel that carries the announcement to everyone)
- The teachers = the event consumers (the parts of the app that listen and respond)

---

## Overview

Event-Driven Architecture (EDA) is a software design pattern where components communicate through the production, detection, and consumption of events. Events represent significant state changes or occurrences in a system, and components react to these events rather than calling each other directly.

This architecture promotes loose coupling, scalability, and real-time responsiveness, making it ideal for complex distributed systems, microservices, and highly interactive applications.

## Core Concepts

### 1. Event System Components

```
┌─────────────────────────────────────────────────────┐
│                  Event Producers                     │
│  (Components that emit events when state changes)   │
└───────────────┬─────────────────────────────────────┘
                │
                ↓
┌─────────────────────────────────────────────────────┐
│                    Event Bus                         │
│  ┌──────────────────────────────────────────────┐  │
│  │  Event Queue / Message Broker                │  │
│  │  - Routing                                    │  │
│  │  - Filtering                                  │  │
│  │  - Persistence                                │  │
│  └──────────────────────────────────────────────┘  │
└───────────────┬─────────────────────────────────────┘
                │
                ↓
┌─────────────────────────────────────────────────────┐
│                 Event Consumers                      │
│  (Components that subscribe to and handle events)   │
└─────────────────────────────────────────────────────┘
```

### 2. Key Patterns

- **Pub/Sub**: Publishers emit events, subscribers receive them
- **Event Sourcing**: Store state changes as a sequence of events
- **CQRS**: Separate read and write models
- **Event Stream**: Process continuous flow of events
- **Saga Pattern**: Coordinate distributed transactions

## Implementation Patterns

### Basic Event Bus (React)

```typescript
// event-bus.ts
type EventHandler<T = any> = (data: T) => void | Promise<void>;

export class EventBus {
  private handlers = new Map<string, Set<EventHandler>>();
  private onceHandlers = new Map<string, Set<EventHandler>>();

  // Subscribe to events
  on<T = any>(event: string, handler: EventHandler<T>): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    
    this.handlers.get(event)!.add(handler);

    // Return unsubscribe function
    return () => {
      this.handlers.get(event)?.delete(handler);
    };
  }

  // Subscribe once
  once<T = any>(event: string, handler: EventHandler<T>): void {
    if (!this.onceHandlers.has(event)) {
      this.onceHandlers.set(event, new Set());
    }
    this.onceHandlers.get(event)!.add(handler);
  }

  // Emit events
  async emit<T = any>(event: string, data?: T): Promise<void> {
    // Handle regular subscribers
    const handlers = this.handlers.get(event);
    if (handlers) {
      const promises = Array.from(handlers).map(handler => 
        Promise.resolve(handler(data))
      );
      await Promise.all(promises);
    }

    // Handle once subscribers
    const onceHandlers = this.onceHandlers.get(event);
    if (onceHandlers) {
      const promises = Array.from(onceHandlers).map(handler =>
        Promise.resolve(handler(data))
      );
      await Promise.all(promises);
      this.onceHandlers.delete(event);
    }
  }

  // Remove all handlers for an event
  off(event: string): void {
    this.handlers.delete(event);
    this.onceHandlers.delete(event);
  }

  // Remove all handlers
  clear(): void {
    this.handlers.clear();
    this.onceHandlers.clear();
  }

  // Get subscriber count
  listenerCount(event: string): number {
    return (this.handlers.get(event)?.size || 0) +
           (this.onceHandlers.get(event)?.size || 0);
  }
}

export const eventBus = new EventBus();

// Usage
eventBus.on('user:login', (user) => {
  console.log('User logged in:', user);
});

eventBus.emit('user:login', { id: 1, name: 'John' });
```

### Typed Event System

```typescript
// typed-events.ts
export interface AppEvents {
  'user:login': { userId: string; timestamp: number };
  'user:logout': { userId: string };
  'order:created': { orderId: string; total: number };
  'order:updated': { orderId: string; status: string };
  'notification:show': { message: string; type: 'info' | 'error' | 'success' };
}

export class TypedEventBus {
  private handlers = new Map<keyof AppEvents, Set<Function>>();

  on<K extends keyof AppEvents>(
    event: K,
    handler: (data: AppEvents[K]) => void | Promise<void>
  ): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    
    this.handlers.get(event)!.add(handler);

    return () => {
      this.handlers.get(event)?.delete(handler);
    };
  }

  async emit<K extends keyof AppEvents>(
    event: K,
    data: AppEvents[K]
  ): Promise<void> {
    const handlers = this.handlers.get(event);
    if (!handlers) return;

    const promises = Array.from(handlers).map(handler =>
      Promise.resolve(handler(data))
    );
    await Promise.all(promises);
  }

  off<K extends keyof AppEvents>(event: K): void {
    this.handlers.delete(event);
  }
}

export const events = new TypedEventBus();

// Type-safe usage
events.on('user:login', (data) => {
  // data is automatically typed as { userId: string; timestamp: number }
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
});

events.emit('user:login', { 
  userId: '123', 
  timestamp: Date.now() 
});
```

### React Event Bus Integration

```typescript
// useEventBus.ts
import { useEffect, useRef } from 'react';
import { eventBus } from './event-bus';

export function useEventBus<T = any>(
  event: string,
  handler: (data: T) => void,
  dependencies: any[] = []
): void {
  const savedHandler = useRef(handler);

  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const eventHandler = (data: T) => savedHandler.current(data);
    const unsubscribe = eventBus.on(event, eventHandler);
    return unsubscribe;
  }, [event, ...dependencies]);
}

export function useEventEmitter() {
  return {
    emit: eventBus.emit.bind(eventBus)
  };
}

// Component usage
function UserProfile() {
  const { emit } = useEventEmitter();

  useEventBus('user:updated', (user) => {
    console.log('User updated:', user);
  });

  const handleSave = async () => {
    // Update user
    await api.updateUser(user);
    
    // Emit event
    emit('user:updated', user);
  };

  return <button onClick={handleSave}>Save</button>;
}

// Another component listening
function UserList() {
  const [users, setUsers] = useState([]);

  useEventBus('user:updated', (updatedUser) => {
    setUsers(prev => 
      prev.map(u => u.id === updatedUser.id ? updatedUser : u)
    );
  });

  return <div>{/* render users */}</div>;
}
```

### Angular Event Bus Service

```typescript
// event-bus.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable, filter, map } from 'rxjs';

export interface AppEvent<T = any> {
  type: string;
  payload: T;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class EventBusService {
  private eventSubject = new Subject<AppEvent>();

  emit<T>(type: string, payload: T): void {
    this.eventSubject.next({
      type,
      payload,
      timestamp: Date.now()
    });
  }

  on<T = any>(type: string): Observable<T> {
    return this.eventSubject.pipe(
      filter(event => event.type === type),
      map(event => event.payload as T)
    );
  }

  onAny(): Observable<AppEvent> {
    return this.eventSubject.asObservable();
  }
}

// Component usage
@Component({
  selector: 'app-user-profile',
  template: `<button (click)="saveUser()">Save</button>`
})
export class UserProfileComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  constructor(private eventBus: EventBusService) {
    // Subscribe to events
    this.eventBus.on<User>('user:updated')
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => {
        console.log('User updated:', user);
      });
  }

  saveUser(): void {
    // Emit event
    this.eventBus.emit('user:updated', this.user);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Event Sourcing Pattern

```typescript
// event-store.ts
export interface DomainEvent {
  id: string;
  type: string;
  aggregateId: string;
  timestamp: number;
  version: number;
  data: any;
}

export class EventStore {
  private events: DomainEvent[] = [];
  private snapshots = new Map<string, any>();

  append(event: Omit<DomainEvent, 'id' | 'timestamp'>): void {
    const domainEvent: DomainEvent = {
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      ...event
    };

    this.events.push(domainEvent);
  }

  getEvents(aggregateId: string, fromVersion = 0): DomainEvent[] {
    return this.events.filter(
      e => e.aggregateId === aggregateId && e.version > fromVersion
    );
  }

  getAllEvents(fromTimestamp = 0): DomainEvent[] {
    return this.events.filter(e => e.timestamp > fromTimestamp);
  }

  saveSnapshot(aggregateId: string, version: number, state: any): void {
    this.snapshots.set(aggregateId, { version, state });
  }

  getSnapshot(aggregateId: string): { version: number; state: any } | null {
    return this.snapshots.get(aggregateId) || null;
  }

  replay<T>(
    aggregateId: string,
    reducer: (state: T, event: DomainEvent) => T,
    initialState: T
  ): T {
    // Try to load from snapshot
    const snapshot = this.getSnapshot(aggregateId);
    let state = snapshot ? snapshot.state : initialState;
    let fromVersion = snapshot ? snapshot.version : 0;

    // Replay events after snapshot
    const events = this.getEvents(aggregateId, fromVersion);
    return events.reduce(reducer, state);
  }
}

// Aggregate example
interface OrderState {
  id: string;
  items: Array<{ productId: string; quantity: number }>;
  status: 'pending' | 'paid' | 'shipped' | 'cancelled';
  total: number;
}

class OrderAggregate {
  constructor(
    private orderId: string,
    private eventStore: EventStore
  ) {}

  // Command handlers
  createOrder(items: any[]): void {
    const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    
    this.eventStore.append({
      type: 'OrderCreated',
      aggregateId: this.orderId,
      version: 1,
      data: { items, total }
    });
  }

  addItem(productId: string, quantity: number): void {
    const state = this.getCurrentState();
    if (state.status !== 'pending') {
      throw new Error('Cannot modify non-pending order');
    }

    this.eventStore.append({
      type: 'ItemAdded',
      aggregateId: this.orderId,
      version: state.items.length + 1,
      data: { productId, quantity }
    });
  }

  pay(): void {
    const state = this.getCurrentState();
    if (state.status !== 'pending') {
      throw new Error('Order already processed');
    }

    this.eventStore.append({
      type: 'OrderPaid',
      aggregateId: this.orderId,
      version: state.items.length + 2,
      data: { paidAt: Date.now() }
    });
  }

  // Event reducer
  private reduce(state: OrderState, event: DomainEvent): OrderState {
    switch (event.type) {
      case 'OrderCreated':
        return {
          id: event.aggregateId,
          items: event.data.items,
          status: 'pending',
          total: event.data.total
        };

      case 'ItemAdded':
        return {
          ...state,
          items: [...state.items, event.data]
        };

      case 'OrderPaid':
        return {
          ...state,
          status: 'paid'
        };

      case 'OrderShipped':
        return {
          ...state,
          status: 'shipped'
        };

      case 'OrderCancelled':
        return {
          ...state,
          status: 'cancelled'
        };

      default:
        return state;
    }
  }

  getCurrentState(): OrderState {
    return this.eventStore.replay(
      this.orderId,
      this.reduce.bind(this),
      { id: this.orderId, items: [], status: 'pending', total: 0 }
    );
  }
}

// Usage
const eventStore = new EventStore();
const order = new OrderAggregate('order-123', eventStore);

order.createOrder([
  { productId: 'p1', quantity: 2, price: 10 }
]);

order.addItem('p2', 1);
order.pay();

const currentState = order.getCurrentState();
console.log(currentState); // { id: 'order-123', items: [...], status: 'paid', ... }
```

### CQRS Pattern

```typescript
// cqrs.ts
export interface Command {
  type: string;
  payload: any;
}

export interface Query {
  type: string;
  params: any;
}

// Command handlers (write side)
export class CommandBus {
  private handlers = new Map<string, (payload: any) => Promise<void>>();

  register(commandType: string, handler: (payload: any) => Promise<void>): void {
    this.handlers.set(commandType, handler);
  }

  async execute(command: Command): Promise<void> {
    const handler = this.handlers.get(command.type);
    if (!handler) {
      throw new Error(`No handler for command: ${command.type}`);
    }
    await handler(command.payload);
  }
}

// Query handlers (read side)
export class QueryBus {
  private handlers = new Map<string, (params: any) => Promise<any>>();

  register(queryType: string, handler: (params: any) => Promise<any>): void {
    this.handlers.set(queryType, handler);
  }

  async execute<T>(query: Query): Promise<T> {
    const handler = this.handlers.get(query.type);
    if (!handler) {
      throw new Error(`No handler for query: ${query.type}`);
    }
    return await handler(query.params);
  }
}

// Write model (commands)
class OrderCommandHandler {
  constructor(
    private eventStore: EventStore,
    private eventBus: EventBus
  ) {}

  async handleCreateOrder(payload: any): Promise<void> {
    // Validate
    if (!payload.items || payload.items.length === 0) {
      throw new Error('Order must have items');
    }

    // Create event
    const orderId = crypto.randomUUID();
    this.eventStore.append({
      type: 'OrderCreated',
      aggregateId: orderId,
      version: 1,
      data: payload
    });

    // Emit event for read model update
    this.eventBus.emit('order:created', { orderId, ...payload });
  }

  async handleCancelOrder(payload: { orderId: string }): Promise<void> {
    // Business logic
    this.eventStore.append({
      type: 'OrderCancelled',
      aggregateId: payload.orderId,
      version: 2,
      data: { cancelledAt: Date.now() }
    });

    this.eventBus.emit('order:cancelled', payload);
  }
}

// Read model (queries)
class OrderReadModel {
  private orders = new Map<string, any>();

  constructor(eventBus: EventBus) {
    // Subscribe to events and update read model
    eventBus.on('order:created', (data) => {
      this.orders.set(data.orderId, {
        id: data.orderId,
        items: data.items,
        status: 'pending',
        createdAt: Date.now()
      });
    });

    eventBus.on('order:cancelled', (data) => {
      const order = this.orders.get(data.orderId);
      if (order) {
        order.status = 'cancelled';
      }
    });
  }

  async getOrder(orderId: string): Promise<any> {
    return this.orders.get(orderId);
  }

  async getAllOrders(): Promise<any[]> {
    return Array.from(this.orders.values());
  }

  async getOrdersByStatus(status: string): Promise<any[]> {
    return Array.from(this.orders.values()).filter(o => o.status === status);
  }
}

// Setup
const eventStore = new EventStore();
const eventBus = new EventBus();
const commandBus = new CommandBus();
const queryBus = new QueryBus();

const commandHandler = new OrderCommandHandler(eventStore, eventBus);
const readModel = new OrderReadModel(eventBus);

commandBus.register('CreateOrder', commandHandler.handleCreateOrder.bind(commandHandler));
commandBus.register('CancelOrder', commandHandler.handleCancelOrder.bind(commandHandler));

queryBus.register('GetOrder', readModel.getOrder.bind(readModel));
queryBus.register('GetAllOrders', readModel.getAllOrders.bind(readModel));

// Usage
await commandBus.execute({
  type: 'CreateOrder',
  payload: { items: [{ productId: 'p1', quantity: 2 }] }
});

const orders = await queryBus.execute({
  type: 'GetAllOrders',
  params: {}
});
```

### Saga Pattern (Distributed Transactions)

```typescript
// saga.ts
export interface SagaStep {
  name: string;
  execute: (context: any) => Promise<any>;
  compensate: (context: any) => Promise<void>;
}

export class Saga {
  private steps: SagaStep[] = [];
  private executedSteps: Array<{ step: SagaStep; result: any }> = [];

  addStep(step: SagaStep): this {
    this.steps.push(step);
    return this;
  }

  async execute(initialContext: any = {}): Promise<any> {
    let context = { ...initialContext };

    try {
      // Execute all steps
      for (const step of this.steps) {
        console.log(`Executing step: ${step.name}`);
        const result = await step.execute(context);
        
        this.executedSteps.push({ step, result });
        context = { ...context, ...result };
      }

      return context;
    } catch (error) {
      console.error('Saga failed, starting compensation:', error);
      await this.compensate();
      throw error;
    }
  }

  private async compensate(): Promise<void> {
    // Compensate in reverse order
    for (const { step, result } of this.executedSteps.reverse()) {
      try {
        console.log(`Compensating step: ${step.name}`);
        await step.compensate(result);
      } catch (error) {
        console.error(`Compensation failed for ${step.name}:`, error);
      }
    }
    this.executedSteps = [];
  }
}

// Order placement saga
const createOrderSaga = new Saga()
  .addStep({
    name: 'Reserve Inventory',
    execute: async (ctx) => {
      const reservation = await inventoryService.reserve(ctx.items);
      return { reservationId: reservation.id };
    },
    compensate: async (ctx) => {
      await inventoryService.cancelReservation(ctx.reservationId);
    }
  })
  .addStep({
    name: 'Process Payment',
    execute: async (ctx) => {
      const payment = await paymentService.charge(ctx.amount);
      return { paymentId: payment.id };
    },
    compensate: async (ctx) => {
      await paymentService.refund(ctx.paymentId);
    }
  })
  .addStep({
    name: 'Create Order',
    execute: async (ctx) => {
      const order = await orderService.create({
        items: ctx.items,
        paymentId: ctx.paymentId,
        reservationId: ctx.reservationId
      });
      return { orderId: order.id };
    },
    compensate: async (ctx) => {
      await orderService.cancel(ctx.orderId);
    }
  })
  .addStep({
    name: 'Send Confirmation',
    execute: async (ctx) => {
      await notificationService.send({
        type: 'order-confirmation',
        orderId: ctx.orderId
      });
      return {};
    },
    compensate: async (ctx) => {
      // Cannot unsend email, just log
      console.log('Sent cancellation email');
    }
  });

// Execute saga
try {
  const result = await createOrderSaga.execute({
    items: [{ productId: 'p1', quantity: 2 }],
    amount: 2000
  });
  console.log('Order created:', result.orderId);
} catch (error) {
  console.error('Order creation failed:', error);
}
```

### Event Stream Processing

```typescript
// event-stream.ts
export class EventStream {
  private stream: DomainEvent[] = [];
  private subscribers = new Set<(event: DomainEvent) => void>();

  append(event: DomainEvent): void {
    this.stream.push(event);
    this.notifySubscribers(event);
  }

  subscribe(handler: (event: DomainEvent) => void): () => void {
    this.subscribers.add(handler);
    return () => this.subscribers.delete(handler);
  }

  private notifySubscribers(event: DomainEvent): void {
    this.subscribers.forEach(handler => handler(event));
  }

  // Stream operations
  filter(predicate: (event: DomainEvent) => boolean): EventStream {
    const filtered = new EventStream();
    this.subscribe(event => {
      if (predicate(event)) {
        filtered.append(event);
      }
    });
    return filtered;
  }

  map<T>(mapper: (event: DomainEvent) => T): T[] {
    return this.stream.map(mapper);
  }

  reduce<T>(reducer: (acc: T, event: DomainEvent) => T, initial: T): T {
    return this.stream.reduce(reducer, initial);
  }

  window(size: number): DomainEvent[][] {
    const windows: DomainEvent[][] = [];
    for (let i = 0; i < this.stream.length; i += size) {
      windows.push(this.stream.slice(i, i + size));
    }
    return windows;
  }
}

// Usage: Process order events
const orderStream = new EventStream();

// Subscribe to specific event types
const orderCreatedStream = orderStream.filter(e => e.type === 'OrderCreated');
orderCreatedStream.subscribe(event => {
  console.log('New order:', event.data);
});

// Calculate metrics
const totalRevenue = orderStream
  .filter(e => e.type === 'OrderPaid')
  .reduce((sum, event) => sum + event.data.amount, 0);

// Process in batches
const batches = orderStream.window(10);
batches.forEach(async batch => {
  await processBatch(batch);
});
```

### Real-time Event Processing with RxJS (Angular)

```typescript
// event-processor.service.ts
import { Injectable } from '@angular/core';
import { 
  Subject, 
  Observable, 
  merge, 
  interval 
} from 'rxjs';
import {
  filter,
  map,
  scan,
  bufferTime,
  groupBy,
  mergeMap,
  debounceTime
} from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class EventProcessorService {
  private eventStream$ = new Subject<DomainEvent>();

  emitEvent(event: DomainEvent): void {
    this.eventStream$.next(event);
  }

  // Get events by type
  getEventsByType(type: string): Observable<DomainEvent> {
    return this.eventStream$.pipe(
      filter(event => event.type === type)
    );
  }

  // Calculate real-time metrics
  getOrderMetrics(): Observable<any> {
    return this.eventStream$.pipe(
      filter(e => e.type === 'OrderCreated'),
      scan((metrics, event) => ({
        count: metrics.count + 1,
        total: metrics.total + event.data.amount,
        average: (metrics.total + event.data.amount) / (metrics.count + 1)
      }), { count: 0, total: 0, average: 0 })
    );
  }

  // Group events by aggregate
  getEventsByAggregate(): Observable<{ aggregateId: string; events: DomainEvent[] }> {
    return this.eventStream$.pipe(
      groupBy(event => event.aggregateId),
      mergeMap(group => 
        group.pipe(
          bufferTime(1000),
          filter(events => events.length > 0),
          map(events => ({
            aggregateId: group.key,
            events
          }))
        )
      )
    );
  }

  // Detect patterns
  detectAnomalies(): Observable<DomainEvent> {
    return this.eventStream$.pipe(
      filter(e => e.type === 'OrderCreated'),
      bufferTime(60000), // 1 minute window
      filter(events => events.length > 100), // More than 100 orders/min
      mergeMap(events => events)
    );
  }

  // Debounce related events
  getDebouncedUserActions(): Observable<DomainEvent> {
    return this.eventStream$.pipe(
      filter(e => e.type.startsWith('user:')),
      debounceTime(500)
    );
  }
}
```

### React Event Stream with Hooks

```typescript
// useEventStream.ts
import { useEffect, useState } from 'react';
import { Subject, Observable } from 'rxjs';
import { filter, map, scan } from 'rxjs/operators';

const eventStream$ = new Subject<DomainEvent>();

export function emitEvent(event: DomainEvent): void {
  eventStream$.next(event);
}

export function useEventStream<T>(
  selector: (stream: Observable<DomainEvent>) => Observable<T>,
  initialValue: T
): T {
  const [value, setValue] = useState<T>(initialValue);

  useEffect(() => {
    const subscription = selector(eventStream$).subscribe(setValue);
    return () => subscription.unsubscribe();
  }, [selector]);

  return value;
}

// Component usage
function OrderMetrics() {
  const metrics = useEventStream(
    stream => stream.pipe(
      filter(e => e.type === 'OrderCreated'),
      scan((acc, event) => ({
        count: acc.count + 1,
        total: acc.total + event.data.amount
      }), { count: 0, total: 0 })
    ),
    { count: 0, total: 0 }
  );

  return (
    <div>
      <p>Total Orders: {metrics.count}</p>
      <p>Total Revenue: ${metrics.total}</p>
    </div>
  );
}
```

### Event Replay and Time Travel

```typescript
// time-travel.ts
export class TimeTravelEventStore extends EventStore {
  private currentPosition = 0;

  getCurrentPosition(): number {
    return this.currentPosition;
  }

  goToPosition(position: number): void {
    if (position < 0 || position > this.events.length) {
      throw new Error('Invalid position');
    }
    this.currentPosition = position;
  }

  goToTimestamp(timestamp: number): void {
    const index = this.events.findIndex(e => e.timestamp >= timestamp);
    this.currentPosition = index === -1 ? this.events.length : index;
  }

  replayTo<T>(
    aggregateId: string,
    reducer: (state: T, event: DomainEvent) => T,
    initialState: T
  ): T {
    const events = this.events
      .filter(e => e.aggregateId === aggregateId)
      .slice(0, this.currentPosition);
    
    return events.reduce(reducer, initialState);
  }

  forward(): DomainEvent | null {
    if (this.currentPosition >= this.events.length) return null;
    return this.events[this.currentPosition++];
  }

  backward(): DomainEvent | null {
    if (this.currentPosition <= 0) return null;
    return this.events[--this.currentPosition];
  }
}

// React time travel debugger
function EventDebugger() {
  const [store] = useState(() => new TimeTravelEventStore());
  const [position, setPosition] = useState(0);
  const [state, setState] = useState<any>(null);

  const goToPosition = (newPosition: number) => {
    store.goToPosition(newPosition);
    setPosition(newPosition);
    
    // Replay to get state at this position
    const currentState = store.replayTo(
      'order-123',
      orderReducer,
      initialOrderState
    );
    setState(currentState);
  };

  return (
    <div>
      <input
        type="range"
        min="0"
        max={store.getAllEvents().length}
        value={position}
        onChange={e => goToPosition(Number(e.target.value))}
      />
      <button onClick={() => goToPosition(position - 1)}>←</button>
      <button onClick={() => goToPosition(position + 1)}>→</button>
      <pre>{JSON.stringify(state, null, 2)}</pre>
    </div>
  );
}
```

## Common Mistakes

### 1. Event Coupling

```typescript
// BAD: Events contain too much data, coupling consumers
eventBus.emit('order:created', {
  order: fullOrderObject,
  user: fullUserObject,
  payment: fullPaymentObject,
  inventory: inventoryDetails
});

// GOOD: Events contain minimal data, consumers fetch what they need
eventBus.emit('order:created', {
  orderId: 'order-123',
  userId: 'user-456',
  timestamp: Date.now()
});

// Consumers fetch additional data if needed
eventBus.on('order:created', async (data) => {
  const order = await orderService.getById(data.orderId);
  // Process order
});
```

### 2. Missing Error Handling

```typescript
// BAD: Errors in one subscriber affect others
eventBus.on('user:login', async (user) => {
  await sendEmail(user); // If this fails, subsequent handlers don't run
  await logAnalytics(user);
  await updateCache(user);
});

// GOOD: Isolate errors
export class RobustEventBus extends EventBus {
  async emit<T>(event: string, data?: T): Promise<void> {
    const handlers = this.handlers.get(event);
    if (!handlers) return;

    const results = await Promise.allSettled(
      Array.from(handlers).map(handler => handler(data))
    );

    // Log failed handlers
    results.forEach((result, index) => {
      if (result.status === 'rejected') {
        console.error(`Handler ${index} failed for ${event}:`, result.reason);
      }
    });
  }
}
```

### 3. Event Versioning Issues

```typescript
// BAD: Breaking changes in events
eventBus.emit('order:created', {
  orderId: 'x',
  items: [...] // Changed structure, breaks old consumers
});

// GOOD: Version events
interface OrderCreatedEventV1 {
  version: 1;
  orderId: string;
  items: Array<{ id: string; qty: number }>;
}

interface OrderCreatedEventV2 {
  version: 2;
  orderId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
}

type OrderCreatedEvent = OrderCreatedEventV1 | OrderCreatedEventV2;

// Handle multiple versions
eventBus.on('order:created', (event: OrderCreatedEvent) => {
  if (event.version === 1) {
    // Handle v1
  } else if (event.version === 2) {
    // Handle v2
  }
});
```

### 4. Memory Leaks

```typescript
// BAD: Subscriptions not cleaned up
useEffect(() => {
  eventBus.on('data:updated', handleUpdate);
  // Missing cleanup
}, []);

// GOOD: Always unsubscribe
useEffect(() => {
  const unsubscribe = eventBus.on('data:updated', handleUpdate);
  return unsubscribe;
}, []);
```

## Best Practices

### 1. Event Naming Convention

```typescript
// Use consistent naming: <entity>:<action>
const eventNames = {
  // User events
  USER_REGISTERED: 'user:registered',
  USER_UPDATED: 'user:updated',
  USER_DELETED: 'user:deleted',
  
  // Order events
  ORDER_CREATED: 'order:created',
  ORDER_PAID: 'order:paid',
  ORDER_SHIPPED: 'order:shipped',
  
  // System events
  SYSTEM_ERROR: 'system:error',
  SYSTEM_READY: 'system:ready'
};
```

### 2. Event Schema Validation

```typescript
import { z } from 'zod';

const orderCreatedSchema = z.object({
  orderId: z.string(),
  userId: z.string(),
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().positive()
  })),
  total: z.number().positive(),
  timestamp: z.number()
});

export class ValidatedEventBus extends EventBus {
  private schemas = new Map<string, z.ZodSchema>();

  registerSchema(event: string, schema: z.ZodSchema): void {
    this.schemas.set(event, schema);
  }

  async emit<T>(event: string, data?: T): Promise<void> {
    const schema = this.schemas.get(event);
    if (schema) {
      try {
        schema.parse(data);
      } catch (error) {
        throw new Error(`Invalid event data for ${event}: ${error}`);
      }
    }
    return super.emit(event, data);
  }
}
```

### 3. Event Logging and Monitoring

```typescript
export class MonitoredEventBus extends EventBus {
  async emit<T>(event: string, data?: T): Promise<void> {
    const startTime = performance.now();
    
    console.log(`[EVENT] Emitting ${event}`, {
      timestamp: new Date().toISOString(),
      subscribers: this.listenerCount(event)
    });

    try {
      await super.emit(event, data);
      
      const duration = performance.now() - startTime;
      console.log(`[EVENT] ${event} completed in ${duration.toFixed(2)}ms`);
    } catch (error) {
      console.error(`[EVENT] ${event} failed:`, error);
      throw error;
    }
  }
}
```

### 4. Event Priority

```typescript
export class PriorityEventBus {
  private handlers = new Map<string, Array<{
    priority: number;
    handler: EventHandler;
  }>>();

  on(event: string, handler: EventHandler, priority = 0): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    
    const handlers = this.handlers.get(event)!;
    handlers.push({ priority, handler });
    
    // Sort by priority (higher first)
    handlers.sort((a, b) => b.priority - a.priority);

    return () => {
      const index = handlers.findIndex(h => h.handler === handler);
      if (index !== -1) handlers.splice(index, 1);
    };
  }

  async emit(event: string, data?: any): Promise<void> {
    const handlers = this.handlers.get(event);
    if (!handlers) return;

    for (const { handler } of handlers) {
      await handler(data);
    }
  }
}

// Usage
const bus = new PriorityEventBus();

bus.on('app:start', () => console.log('Low priority'), 0);
bus.on('app:start', () => console.log('High priority'), 10);
bus.on('app:start', () => console.log('Critical'), 100);

bus.emit('app:start');
// Output:
// Critical
// High priority
// Low priority
```

## When to Use Event-Driven Architecture

### Use When

1. **Loose Coupling**: Components should be independent
2. **Scalability**: System needs to scale components independently
3. **Real-time**: Need real-time updates across components
4. **Audit Trail**: Need complete history of state changes
5. **Integration**: Multiple systems need to react to changes
6. **Async Processing**: Work can be processed asynchronously

### Don't Use When

1. **Simple Apps**: Small applications with few interactions
2. **Strong Consistency**: Need immediate consistency across system
3. **Simple Flow**: Linear, predictable control flow
4. **Debugging Priority**: Need simple call stacks for debugging
5. **Team Unfamiliarity**: Team not comfortable with async patterns

## Interview Questions

### Q1: What's the difference between Event-Driven Architecture and Observer pattern?

Answer: EDA is an architectural pattern for system-wide communication, while Observer is a design pattern for object-to-object communication. EDA typically involves message brokers, persistence, and distributed systems. Observer is synchronous and in-process.

### Q2: How do you handle eventual consistency in event-driven systems?

Answer: Use techniques like event sourcing for complete history, CQRS to separate reads/writes, sagas for distributed transactions, idempotent consumers to handle duplicate events, and compensation logic for failures.

### Q3: What's the difference between Event Sourcing and Event-Driven Architecture?

Answer: Event-Driven Architecture is about communication between components via events. Event Sourcing is about storing state as a sequence of events. You can have EDA without Event Sourcing and vice versa, but they work well together.

### Q4: How do you handle failed event processing?

Answer: Implement retry logic with exponential backoff, use dead letter queues for persistent failures, log failures for investigation, implement circuit breakers to prevent cascade failures, and design idempotent handlers.

### Q5: How do you debug event-driven systems?

Answer: Use comprehensive logging with correlation IDs, implement event tracing across services, use event replay for debugging, maintain event schemas and documentation, and use monitoring tools to visualize event flow.

## Key Takeaways

1. **Event-Driven Architecture promotes loose coupling** through asynchronous communication
2. **Events represent facts** - things that happened in the system
3. **Use Event Sourcing** for complete audit trail and time travel
4. **CQRS separates reads and writes** for scalability
5. **Sagas coordinate distributed transactions** with compensation logic
6. **Handle failures gracefully** - one subscriber failure shouldn't affect others
7. **Version events** to handle schema evolution
8. **Monitor and log events** for debugging and observability
9. **Events should be immutable** - never modify after emission
10. **Clean up subscriptions** to prevent memory leaks

## Resources

### Documentation
- [Event-Driven Architecture (Martin Fowler)](https://martinfowler.com/articles/201701-event-driven.html)
- [Event Sourcing Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [RxJS Documentation](https://rxjs.dev/)

### Tools
- [RxJS](https://rxjs.dev/) - Reactive programming library
- [EventEmitter (Node.js)](https://nodejs.org/api/events.html)
- [Redux](https://redux.js.org/) - State management with events
- [MobX](https://mobx.js.org/) - Reactive state management

### Books
- "Enterprise Integration Patterns" by Gregor Hohpe
- "Designing Event-Driven Systems" by Ben Stopford
- "Microservices Patterns" by Chris Richardson
- "Building Event-Driven Microservices" by Adam Bellemare
