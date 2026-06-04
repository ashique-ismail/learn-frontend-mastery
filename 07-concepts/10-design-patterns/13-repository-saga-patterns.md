# Repository and Saga Patterns

## Overview

The Repository pattern abstracts data access behind a consistent interface, decoupling business logic from the specifics of APIs, databases, and storage mechanisms. The Saga pattern orchestrates complex async workflows — multi-step processes with compensation logic (rollback on failure). Both patterns address different layers of a frontend application: Repository lives at the data access layer; Saga lives at the workflow orchestration layer. This guide covers both with practical TypeScript implementations and their relationship to state management.

## Repository Pattern

### Problem: Scattered Data Access

```typescript
// ❌ Business logic entangled with HTTP details
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    // HTTP details mixed into component
    fetch(`/api/users/${userId}`, {
      headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
    })
    .then(r => {
      if (!r.ok) throw new Error('Failed to fetch user');
      return r.json();
    })
    .then(data => setUser(data))
    .catch(console.error);
  }, [userId]);
}

// Problems:
// - Can't swap REST for GraphQL without changing every component
// - Can't test without mocking fetch
// - Can't add caching without touching business logic
// - Auth header logic duplicated everywhere
```

### Repository Interface

```typescript
// Define the contract — independent of implementation
interface UserRepository {
  findById(id: string): Promise<User>;
  findByEmail(email: string): Promise<User | null>;
  create(data: CreateUserInput): Promise<User>;
  update(id: string, data: UpdateUserInput): Promise<User>;
  delete(id: string): Promise<void>;
  list(params: ListUsersParams): Promise<{ users: User[]; total: number; nextCursor?: string }>;
}

interface PostRepository {
  findById(id: string): Promise<Post>;
  findByAuthor(authorId: string, params?: PaginationParams): Promise<Post[]>;
  create(data: CreatePostInput): Promise<Post>;
  update(id: string, data: UpdatePostInput): Promise<Post>;
  publish(id: string): Promise<Post>;
  delete(id: string): Promise<void>;
  search(query: string): Promise<Post[]>;
}
```

### REST Implementation

```typescript
// REST API implementation
class RestUserRepository implements UserRepository {
  constructor(private httpClient: HttpClient) {}

  async findById(id: string): Promise<User> {
    const response = await this.httpClient.get<User>(`/users/${id}`);
    return response.data;
  }

  async findByEmail(email: string): Promise<User | null> {
    try {
      const response = await this.httpClient.get<User>(`/users/by-email/${encodeURIComponent(email)}`);
      return response.data;
    } catch (err) {
      if (err instanceof HttpError && err.status === 404) return null;
      throw err;
    }
  }

  async create(data: CreateUserInput): Promise<User> {
    const response = await this.httpClient.post<User>('/users', data);
    return response.data;
  }

  async update(id: string, data: UpdateUserInput): Promise<User> {
    const response = await this.httpClient.patch<User>(`/users/${id}`, data);
    return response.data;
  }

  async delete(id: string): Promise<void> {
    await this.httpClient.delete(`/users/${id}`);
  }

  async list(params: ListUsersParams): Promise<{ users: User[]; total: number; nextCursor?: string }> {
    const response = await this.httpClient.get<{ users: User[]; total: number; next?: string }>(
      '/users',
      { params }
    );
    return {
      users: response.data.users,
      total: response.data.total,
      nextCursor: response.data.next,
    };
  }
}
```

### Cached Repository (Decorator Pattern)

```typescript
// Wrap any repository with caching — no changes to RestUserRepository
class CachedUserRepository implements UserRepository {
  private cache = new Map<string, { data: User; expiresAt: number }>();
  private TTL = 5 * 60 * 1000; // 5 minutes

  constructor(private repository: UserRepository) {}

  async findById(id: string): Promise<User> {
    const cached = this.cache.get(`user:${id}`);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.data;
    }

    const user = await this.repository.findById(id);
    this.cache.set(`user:${id}`, { data: user, expiresAt: Date.now() + this.TTL });
    return user;
  }

  async update(id: string, data: UpdateUserInput): Promise<User> {
    const user = await this.repository.update(id, data);
    // Invalidate cache on update
    this.cache.delete(`user:${id}`);
    return user;
  }

  async delete(id: string): Promise<void> {
    await this.repository.delete(id);
    this.cache.delete(`user:${id}`);
  }

  // Delegate non-cached methods
  findByEmail = this.repository.findByEmail.bind(this.repository);
  create = this.repository.create.bind(this.repository);
  list = this.repository.list.bind(this.repository);
}
```

### In-Memory Repository for Testing

```typescript
// Test double — no HTTP, no network, fully controlled
class InMemoryUserRepository implements UserRepository {
  private users = new Map<string, User>();

  constructor(initialUsers: User[] = []) {
    initialUsers.forEach(u => this.users.set(u.id, u));
  }

  async findById(id: string): Promise<User> {
    const user = this.users.get(id);
    if (!user) throw new NotFoundError(`User ${id} not found`);
    return { ...user };
  }

  async findByEmail(email: string): Promise<User | null> {
    const user = [...this.users.values()].find(u => u.email === email);
    return user ? { ...user } : null;
  }

  async create(data: CreateUserInput): Promise<User> {
    const user: User = { id: generateId(), createdAt: new Date(), ...data };
    this.users.set(user.id, user);
    return { ...user };
  }

  async update(id: string, data: UpdateUserInput): Promise<User> {
    const existing = await this.findById(id);
    const updated = { ...existing, ...data };
    this.users.set(id, updated);
    return { ...updated };
  }

  async delete(id: string): Promise<void> {
    if (!this.users.has(id)) throw new NotFoundError(`User ${id} not found`);
    this.users.delete(id);
  }

  async list(params: ListUsersParams): Promise<{ users: User[]; total: number }> {
    const all = [...this.users.values()];
    return { users: all, total: all.length };
  }
}

// In tests — no mocking needed
test('updates user profile', async () => {
  const repo = new InMemoryUserRepository([
    { id: 'u1', name: 'Alice', email: 'alice@example.com', role: 'viewer' },
  ]);
  const service = new UserService(repo);

  const updated = await service.updateProfile('u1', { name: 'Alice Smith' });
  expect(updated.name).toBe('Alice Smith');
});
```

### Dependency Injection

```typescript
// Compose repositories and inject into services/hooks
class UserService {
  constructor(private users: UserRepository, private posts: PostRepository) {}

  async deleteUserAndPosts(userId: string): Promise<void> {
    const posts = await this.posts.findByAuthor(userId);
    await Promise.all(posts.map(p => this.posts.delete(p.id)));
    await this.users.delete(userId);
  }
}

// Production
const userRepo = new CachedUserRepository(new RestUserRepository(httpClient));
const postRepo = new RestPostRepository(httpClient);
const userService = new UserService(userRepo, postRepo);

// Testing
const testUserRepo = new InMemoryUserRepository([testUser]);
const testPostRepo = new InMemoryPostRepository([testPost]);
const testService = new UserService(testUserRepo, testPostRepo);
```

## Saga Pattern

### Problem: Complex Async Workflows

```typescript
// ❌ Complex async workflow with error handling — spaghetti
async function checkout(cartId: string) {
  try {
    const order = await createOrder(cartId);
    try {
      const payment = await processPayment(order.id);
      try {
        await reserveInventory(order.items);
        try {
          await sendConfirmationEmail(order.id);
          await clearCart(cartId);
          return order;
        } catch (emailError) {
          // Email failed — what do we do? We've already charged the customer!
          console.error('Email failed but order succeeded:', emailError);
          return order; // Continue despite email failure?
        }
      } catch (inventoryError) {
        // Inventory failed — must refund payment!
        await refundPayment(payment.id);
        throw inventoryError;
      }
    } catch (paymentError) {
      // Payment failed — must cancel order
      await cancelOrder(order.id);
      throw paymentError;
    }
  } catch (orderError) {
    throw orderError;
  }
}
```

### Saga with Compensation

A Saga is a sequence of steps where each step has a compensating action (rollback):

```typescript
interface SagaStep<TState> {
  name: string;
  execute(state: TState): Promise<Partial<TState>>;
  compensate(state: TState): Promise<void>;
}

class Saga<TState extends Record<string, unknown>> {
  private steps: SagaStep<TState>[] = [];
  private completedSteps: Array<SagaStep<TState>> = [];

  addStep(step: SagaStep<TState>): this {
    this.steps.push(step);
    return this;
  }

  async execute(initialState: TState): Promise<TState> {
    let state = { ...initialState };
    this.completedSteps = [];

    for (const step of this.steps) {
      try {
        console.log(`Executing saga step: ${step.name}`);
        const result = await step.execute(state);
        state = { ...state, ...result };
        this.completedSteps.push(step);
      } catch (error) {
        console.error(`Saga step "${step.name}" failed — compensating...`);
        await this.compensate(state);
        throw error;
      }
    }

    return state;
  }

  private async compensate(state: TState): Promise<void> {
    // Compensate in reverse order — undo last first
    const toCompensate = [...this.completedSteps].reverse();
    for (const step of toCompensate) {
      try {
        await step.compensate(state);
        console.log(`Compensated: ${step.name}`);
      } catch (err) {
        console.error(`Compensation failed for "${step.name}":`, err);
        // Log but continue — best-effort compensation
      }
    }
  }
}

// Checkout saga — clean definition
interface CheckoutState {
  cartId: string;
  orderId?: string;
  paymentId?: string;
  inventoryReserved?: boolean;
}

const checkoutSaga = new Saga<CheckoutState>()
  .addStep({
    name: 'Create Order',
    async execute(state) {
      const order = await orderService.create(state.cartId);
      return { orderId: order.id };
    },
    async compensate(state) {
      if (state.orderId) await orderService.cancel(state.orderId);
    },
  })
  .addStep({
    name: 'Process Payment',
    async execute(state) {
      const payment = await paymentService.charge(state.orderId!);
      return { paymentId: payment.id };
    },
    async compensate(state) {
      if (state.paymentId) await paymentService.refund(state.paymentId);
    },
  })
  .addStep({
    name: 'Reserve Inventory',
    async execute(state) {
      await inventoryService.reserve(state.orderId!);
      return { inventoryReserved: true };
    },
    async compensate(state) {
      if (state.inventoryReserved) await inventoryService.release(state.orderId!);
    },
  })
  .addStep({
    name: 'Send Confirmation',
    async execute(state) {
      await emailService.sendOrderConfirmation(state.orderId!);
      return {}; // no state to update
    },
    async compensate() {
      // Emails can't be unsent — no compensation possible
      // Log it and move on
    },
  });

// Usage
const result = await checkoutSaga.execute({ cartId: 'c1' });
// If payment fails: createOrder is compensated (cancelled)
// If inventory fails: payment is refunded, order is cancelled
```

## Redux-Saga

`redux-saga` is a Redux middleware for managing side effects using generator functions:

```typescript
import { call, put, takeEvery, select, all, fork } from 'redux-saga/effects';

// Generator-based saga — each yield is a step
function* checkoutSaga(action: CheckoutAction) {
  try {
    // Step 1: Create order
    const order: Order = yield call(orderApi.create, action.payload.cartId);
    yield put(orderCreated(order));

    // Step 2: Process payment
    const payment: Payment = yield call(paymentApi.charge, {
      orderId: order.id,
      amount: order.total,
    });
    yield put(paymentProcessed(payment));

    // Step 3: Reserve inventory
    yield call(inventoryApi.reserve, order.id);

    // Step 4: Confirm
    yield put(checkoutCompleted({ orderId: order.id }));
    yield call(emailApi.sendConfirmation, order.id);
  } catch (error) {
    // Compensation based on what succeeded
    const { orderId, paymentId } = yield select(selectCheckoutState);

    if (paymentId) yield call(paymentApi.refund, paymentId);
    if (orderId) yield call(orderApi.cancel, orderId);

    yield put(checkoutFailed(error as Error));
  }
}

// Watch for checkout actions
function* watchCheckout() {
  yield takeEvery(CHECKOUT_INITIATED, checkoutSaga);
}

// Root saga
function* rootSaga() {
  yield all([
    fork(watchCheckout),
    fork(watchUserEvents),
    fork(watchCartEvents),
  ]);
}
```

## XState — State Machine Sagas

XState models sagas as explicit state machines:

```typescript
import { createMachine, assign } from 'xstate';

const checkoutMachine = createMachine({
  id: 'checkout',
  initial: 'idle',
  context: { orderId: null, paymentId: null, error: null },
  states: {
    idle: {
      on: { INITIATE: 'creatingOrder' },
    },
    creatingOrder: {
      invoke: {
        src: (ctx) => orderApi.create(ctx.cartId),
        onDone: {
          target: 'processingPayment',
          actions: assign({ orderId: (_, event) => event.data.id }),
        },
        onError: {
          target: 'failed',
          actions: assign({ error: (_, event) => event.data }),
        },
      },
    },
    processingPayment: {
      invoke: {
        src: (ctx) => paymentApi.charge(ctx.orderId),
        onDone: {
          target: 'reservingInventory',
          actions: assign({ paymentId: (_, event) => event.data.id }),
        },
        onError: {
          target: 'compensatingPayment',
          actions: assign({ error: (_, event) => event.data }),
        },
      },
    },
    // ... more states
    compensatingPayment: {
      invoke: {
        src: (ctx) => orderApi.cancel(ctx.orderId),
        onDone: 'failed',
        onError: 'failed', // best-effort
      },
    },
    completed: { type: 'final' },
    failed: { type: 'final' },
  },
});
```

## Common Mistakes

### 1. Repository That Returns Framework Objects Instead of Domain Objects

```typescript
// ❌ Repository returns Axios response — exposes HTTP layer to domain
async findById(id: string): Promise<AxiosResponse<User>> {
  return this.httpClient.get(`/users/${id}`);
}

// ✅ Repository returns domain objects
async findById(id: string): Promise<User> {
  const response = await this.httpClient.get<User>(`/users/${id}`);
  return response.data; // unwrap — return domain object
}
```

### 2. Saga with Missing Compensation Steps

```typescript
// ❌ No compensation for payment if inventory fails
addStep({
  name: 'Reserve Inventory',
  async execute(state) { await inventoryService.reserve(state.orderId!); },
  async compensate() { /* missing! */ }, // ← bug: payment not refunded on inventory failure
});

// ✅ Every step with observable effects needs compensation
addStep({
  name: 'Reserve Inventory',
  async execute(state) {
    await inventoryService.reserve(state.orderId!);
    return { inventoryReserved: true };
  },
  async compensate(state) {
    if (state.inventoryReserved) {
      await inventoryService.release(state.orderId!);
    }
  },
});
```

## Interview Questions

### 1. What problem does the Repository pattern solve and how does it differ from a service layer?

**Answer:** The Repository pattern abstracts data access behind a stable interface, decoupling business logic from the specifics of HTTP, GraphQL, IndexedDB, or any storage mechanism. If the API changes from REST to GraphQL, only the repository implementation changes — not the components or services that use it. A Service layer contains business logic and orchestration (calling multiple repositories, applying business rules). A Repository only handles data access for one entity type — no business rules. The distinction: Repository is about how data is fetched/stored; Service is about what to do with the data.

### 2. How does the Repository pattern improve testability?

**Answer:** By defining the repository as an interface, you can create an `InMemoryRepository` that satisfies the same contract using a Map or array instead of HTTP calls. Tests instantiate the service with the in-memory repository — no mocking frameworks, no HTTP interception, no test servers needed. Tests are fast, deterministic, and readable. The in-memory implementation is also a valid specification of the contract — if both the in-memory and REST implementations pass the same test suite, you can be confident they're equivalent.

### 3. What is the Saga pattern and when would you use it over a simple try/catch chain?

**Answer:** The Saga pattern manages complex multi-step workflows where each step has a compensating action (rollback) in case of failure. It's appropriate when a workflow spans multiple services or resources and partial failure would leave data in an inconsistent state — a payment charged but inventory not reserved, or an order created but not paid for. Simple try/catch works for one or two steps but grows unwieldy with 5+ steps where each failure requires different rollback logic. The Saga formalizes this into a reusable structure: an ordered list of (execute, compensate) pairs that the framework executes forward, and on failure, compensates in reverse order.

### 4. How does redux-saga differ from async thunks?

**Answer:** Redux-Thunk dispatches a function that runs imperatively, mixing async logic with dispatch calls directly. Redux-Saga runs in a separate thread (generator coroutine) that yields declarative "effects" — `call()`, `put()`, `select()` — rather than executing them directly. This makes sagas testable by yielding effect descriptors that tests can check without running real async code. Sagas support more advanced patterns: `takeLatest` (cancel previous if new action arrives), `race` (first of N async operations wins), `all` (parallel execution), and `channel` patterns. The cost is complexity — generators and saga effects have a steeper learning curve than thunks.
