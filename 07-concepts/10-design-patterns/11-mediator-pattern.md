# Mediator Pattern

## The Idea

**In plain English:** The Mediator pattern is a way of organizing communication in a program so that separate pieces (called components) never talk to each other directly — instead, they all send messages to one central middleman (the mediator), who figures out who needs to hear what.

**Real-world analogy:** Think of an air traffic control tower at a busy airport. Planes never radio each other directly to coordinate landing and takeoff — they all talk only to the control tower, which decides who goes when and where.

- The control tower = the mediator (the central coordinator in code)
- Each airplane = a component (a self-contained piece of the program)
- A radio call from a plane = an event or message sent to the mediator
- The tower's instructions back to a plane = the mediator triggering an action on a component

---

## Overview

The Mediator pattern introduces a central coordinator (the mediator) that handles communication between components, preventing them from referring to each other directly. Without a mediator, N components communicating directly with each other requires up to N*(N-1) connections. With a mediator, each component knows only the mediator — N connections total. This reduces coupling dramatically, makes the communication flow visible in one place, and makes components independently testable. In frontend applications, the most common mediator implementations are event buses, pub-sub systems, and centralized state managers.

## The Problem: Direct Component Coupling

```
Without Mediator (N*(N-1) connections):

  ComponentA ←→ ComponentB
      ↑ ↓          ↑ ↓
  ComponentD ←→ ComponentC

  Each component knows about ALL others.
  Adding ComponentE requires updating 4 existing components.
  Testing ComponentA requires mocking B, C, D.

With Mediator (N connections):

  ComponentA  ComponentB
      ↓ ↑         ↓ ↑
      └─── [MEDIATOR] ───┘
      ↑ ↓         ↑ ↓
  ComponentD  ComponentC

  Each component knows only the mediator.
  Adding ComponentE requires adding 1 connection.
  Testing ComponentA requires only mocking the mediator.
```

## Basic Mediator

```typescript
// Mediator interface
interface Mediator {
  notify(sender: Component, event: string, data?: unknown): void;
}

// Abstract component — knows about the mediator, not other components
abstract class Component {
  constructor(protected mediator: Mediator) {}

  send(event: string, data?: unknown): void {
    this.mediator.notify(this, event, data);
  }
}

// Concrete components — know NOTHING about each other
class LoginForm extends Component {
  submit(credentials: { email: string; password: string }): void {
    // Instead of calling AuthService or NavigationService directly:
    this.send('LOGIN_SUBMITTED', credentials);
  }

  showError(message: string): void {
    console.log(`Login error: ${message}`);
  }
}

class Dashboard extends Component {
  displayUserData(user: User): void {
    console.log(`Welcome, ${user.name}`);
  }
}

class Notification extends Component {
  show(message: string): void {
    console.log(`Notification: ${message}`);
  }
}

// Concrete mediator — coordinates all interactions
class AuthMediator implements Mediator {
  constructor(
    private loginForm: LoginForm,
    private dashboard: Dashboard,
    private notification: Notification,
    private authService: AuthService
  ) {}

  async notify(sender: Component, event: string, data?: unknown): Promise<void> {
    if (event === 'LOGIN_SUBMITTED') {
      const credentials = data as { email: string; password: string };
      try {
        const user = await this.authService.login(credentials);
        this.dashboard.displayUserData(user);
        this.notification.show(`Welcome back, ${user.name}!`);
      } catch (err) {
        this.loginForm.showError('Invalid credentials');
      }
    }
  }
}
```

## Event Bus — The Most Common Frontend Mediator

An event bus is a publish/subscribe implementation — components publish events without knowing who will receive them, and subscribers listen without knowing who published:

```typescript
type EventHandler<T = unknown> = (data: T) => void;

class EventBus {
  private handlers = new Map<string, Set<EventHandler>>();

  on<T>(event: string, handler: EventHandler<T>): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler as EventHandler);

    // Return unsubscribe function
    return () => this.handlers.get(event)?.delete(handler as EventHandler);
  }

  once<T>(event: string, handler: EventHandler<T>): void {
    const unsubscribe = this.on<T>(event, (data) => {
      handler(data);
      unsubscribe();
    });
  }

  emit<T>(event: string, data?: T): void {
    const handlers = this.handlers.get(event);
    if (handlers) {
      handlers.forEach((handler) => {
        try {
          handler(data);
        } catch (err) {
          console.error(`Error in handler for event "${event}":`, err);
        }
      });
    }
  }

  off(event: string): void {
    this.handlers.delete(event);
  }

  clear(): void {
    this.handlers.clear();
  }
}

// Singleton event bus (application-wide mediator)
export const eventBus = new EventBus();
```

### Typed Event Bus

```typescript
// TypeScript typed event bus — events and payloads are type-checked
interface AppEvents {
  'user:login': { userId: string; email: string };
  'user:logout': void;
  'cart:item-added': { productId: string; quantity: number };
  'cart:checkout-started': void;
  'notification:show': { message: string; type: 'info' | 'error' | 'success' };
}

class TypedEventBus {
  private handlers = new Map<string, Set<Function>>();

  on<K extends keyof AppEvents>(
    event: K,
    handler: AppEvents[K] extends void ? () => void : (data: AppEvents[K]) => void
  ): () => void {
    if (!this.handlers.has(event as string)) {
      this.handlers.set(event as string, new Set());
    }
    this.handlers.get(event as string)!.add(handler);
    return () => this.handlers.get(event as string)?.delete(handler);
  }

  emit<K extends keyof AppEvents>(
    ...args: AppEvents[K] extends void ? [event: K] : [event: K, data: AppEvents[K]]
  ): void {
    const [event, data] = args;
    this.handlers.get(event as string)?.forEach((h) => h(data));
  }
}

const bus = new TypedEventBus();

// ✅ Fully typed — IDE autocomplete on event names AND payload types
bus.on('user:login', ({ userId, email }) => {
  console.log(`User ${userId} (${email}) logged in`);
});

bus.emit('user:login', { userId: 'u1', email: 'alice@example.com' });
bus.emit('cart:item-added', { productId: 'p1', quantity: 2 });
// bus.emit('user:login', { wrong: 'shape' }); // TypeScript error
```

## React Event Bus Integration

```typescript
// React hook for event bus subscriptions
import { useEffect, useCallback } from 'react';
import { eventBus, AppEvents } from './eventBus';

function useEventBus<K extends keyof AppEvents>(
  event: K,
  handler: AppEvents[K] extends void ? () => void : (data: AppEvents[K]) => void,
  deps: React.DependencyList = []
): void {
  const stableHandler = useCallback(handler as Function, deps);

  useEffect(() => {
    const unsubscribe = eventBus.on(event, stableHandler as any);
    return unsubscribe; // cleanup on unmount
  }, [event, stableHandler]);
}

// Components communicate without knowing each other
function CartButton({ productId }: { productId: string }) {
  const handleClick = () => {
    eventBus.emit('cart:item-added', { productId, quantity: 1 });
    // CartButton knows nothing about CartBadge or CartDrawer
  };

  return <button onClick={handleClick}>Add to Cart</button>;
}

function CartBadge() {
  const [count, setCount] = useState(0);

  useEventBus('cart:item-added', ({ quantity }) => {
    setCount((c) => c + quantity);
  });

  return <span className="cart-badge">{count}</span>;
}

function CartDrawer() {
  const [isOpen, setIsOpen] = useState(false);

  useEventBus('cart:item-added', () => {
    setIsOpen(true); // open drawer when item added
  });

  return isOpen ? <DrawerContent /> : null;
}
```

## Mediator vs Observer

```
Mediator vs Observer:

Observer (pub-sub):
  Publisher knows about subscribers (directly or via observable)
  Subject manages its own list of observers
  Communication: subject → observers

  Observable state:
  state$ → [subscriber1, subscriber2, subscriber3]

Mediator (event bus):
  Publisher doesn't know about subscribers (decoupled)
  Central mediator manages all subscriptions
  Communication: component → mediator → component

  EventBus:
  ComponentA → emit('event') → [handler1, handler2]

Distinction is subtle — event bus IS a mediator using pub-sub internally.
The key difference: in Observer, the subject manages its observers.
In Mediator, the central bus manages all routing.
```

## Mediator in Angular

Angular's `EventEmitter`, `Subject` (RxJS), and services all serve as mediators:

```typescript
// Angular shared service as mediator
@Injectable({ providedIn: 'root' })
export class CartMediatorService {
  private itemAdded$ = new Subject<CartItem>();
  private checkout$ = new Subject<void>();
  private cartCleared$ = new Subject<void>();

  // Streams for different components to subscribe to
  get onItemAdded(): Observable<CartItem> {
    return this.itemAdded$.asObservable();
  }

  get onCheckout(): Observable<void> {
    return this.checkout$.asObservable();
  }

  // Components call these methods — they don't know who listens
  addItem(item: CartItem): void {
    this.itemAdded$.next(item);
  }

  initiateCheckout(): void {
    this.checkout$.next();
  }
}

// Component A — adds items, knows nothing about CartBadge
@Component({...})
export class ProductCardComponent {
  constructor(private cartMediator: CartMediatorService) {}

  addToCart(product: Product): void {
    this.cartMediator.addItem({ productId: product.id, quantity: 1, price: product.price });
  }
}

// Component B — shows badge count, knows nothing about ProductCard
@Component({...})
export class CartBadgeComponent implements OnInit, OnDestroy {
  count = 0;
  private subscription: Subscription;

  constructor(private cartMediator: CartMediatorService) {}

  ngOnInit(): void {
    this.subscription = this.cartMediator.onItemAdded.subscribe(({ quantity }) => {
      this.count += quantity;
    });
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }
}
```

## Common Mistakes

### 1. Making the Mediator a God Object

```typescript
// ❌ Mediator knows and coordinates EVERYTHING — becomes god object
class AppMediator {
  handleEvent(event: string, data: unknown): void {
    if (event === 'USER_LOGIN') { /* 50 lines */ }
    if (event === 'USER_LOGOUT') { /* 30 lines */ }
    if (event === 'CART_ADD') { /* 40 lines */ }
    if (event === 'ORDER_PLACED') { /* 60 lines */ }
    // ... 20 more if blocks
  }
}

// ✅ Separate mediators per domain, or use a typed event bus
// where each subscriber handles its own domain
```

### 2. Memory Leaks from Unremoved Listeners

```typescript
// ❌ Component adds listener but never removes it
function UserProfile() {
  useEffect(() => {
    eventBus.on('user:updated', updateProfile); // never unsubscribed!
  }, []);
}

// ✅ Return cleanup function from useEffect
function UserProfile() {
  useEffect(() => {
    const unsubscribe = eventBus.on('user:updated', updateProfile);
    return unsubscribe; // removed when component unmounts
  }, []);
}
```

### 3. Overusing Event Bus for Direct Communication

```typescript
// ❌ Using event bus for parent-child communication
// Parent emits event; child listens — but they're directly related!
function Parent() {
  const handleClick = () => eventBus.emit('button:clicked'); // unnecessary indirection
  return <Child />;
}

function Child() {
  useEventBus('button:clicked', doSomething);
}

// ✅ Props for direct parent-child relationships
function Parent() {
  return <Child onButtonClick={doSomething} />;
}
```

## Interview Questions

### 1. What is the Mediator pattern and how does it differ from direct component coupling?

**Answer:** The Mediator introduces a central coordinator that handles all communication between components. Without it, N components communicating pairwise need up to N*(N-1) connections — each component must know about all others it talks to. With a mediator, each component only knows the mediator — N connections total. This reduces coupling (components are independently reusable without their "conversation partners"), centralizes communication logic (the flow is visible in the mediator), and makes components testable in isolation (mock only the mediator, not all collaborators).

### 2. What is the difference between the Mediator and Observer patterns?

**Answer:** In Observer, the subject (Observable) maintains its own list of observers and notifies them directly — the subject knows about its observers. In Mediator, no component knows about any other component; they all communicate through a central mediator. An event bus is technically a Mediator that uses pub-sub internally — but the Publisher doesn't manage subscribers (the bus does). The practical distinction: Observer is about one object notifying others of its own state changes; Mediator is about decoupling objects that need to communicate where neither side knows the other.

### 3. When would you choose an event bus over React Context or Redux for cross-component communication?

**Answer:** Event bus for fire-and-forget notifications — "user added item to cart" that triggers multiple side effects across unrelated components (badge update, drawer open, analytics event) without a shared state object. React Context for shared state that components read — theme, locale, current user. Redux for complex state with time-travel debugging, derived state (selectors), and predictable update flow. The rule of thumb: if you need to store and read the value, use Context or Redux. If you're broadcasting that something happened and subscribers react in their own way, an event bus is simpler. Overusing event buses for state leads to the same problems as prop drilling — the state is implicit and hard to trace.

### 4. How do you prevent memory leaks with event bus subscriptions in React?

**Answer:** Every `eventBus.on()` call must have a corresponding cleanup. In React, the `useEffect` hook's return value is the cleanup function — return the unsubscribe function returned by `eventBus.on()`. Create a `useEventBus` hook that encapsulates this pattern so components can't forget the cleanup. In Angular, store subscriptions in a `Subscription` object and call `.unsubscribe()` in `ngOnDestroy`. For class-based code, track subscriptions in an array and unsubscribe all in the destructor. The symptom of missing cleanup is a component that has been unmounted but still receives and processes events, often causing state updates on unmounted components.
