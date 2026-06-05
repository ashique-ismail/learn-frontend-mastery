# Effect-TS

## The Idea

**In plain English:** Effect-TS is a TypeScript library that lets you describe what your program should do — including what can go wrong — before actually running it. Instead of code that immediately fires off and throws surprise errors, you build up a blueprint of steps, errors, and dependencies that TypeScript can check and reason about ahead of time.

**Real-world analogy:** Imagine a restaurant kitchen using printed order tickets. The waiter writes down the full order — including allergy notes and substitutions — on a ticket before anything is cooked. The kitchen reads the ticket, knows exactly what could go wrong (no avocado in stock, oven down), and only then starts cooking.

- The order ticket = the Effect (a description of the computation, not yet executed)
- The allergy notes = typed errors (known failure cases baked into the description)
- The kitchen staff = the runtime that actually executes the Effect when you call `Effect.runPromise`

---

## What It Is

Effect is a TypeScript library for building robust, type-safe, scalable applications. It models computations as values, provides typed error handling (no thrown exceptions), structured concurrency, dependency injection, and resource management — all composable and highly testable.

Think of it as: **fp-ts + async/await + structured concurrency + DI + telemetry** in one coherent package.

---

## The Core Type: `Effect<Success, Error, Requirements>`

```ts
import { Effect } from 'effect';

// Effect<User, UserNotFound, never>
// Success: User | Error: UserNotFound | Requirements: none
const fetchUser = (id: string): Effect.Effect<User, UserNotFound, never> =>
  Effect.tryPromise({
    try: () => api.getUser(id),
    catch: (error) => new UserNotFound({ id }),
  });

// The Effect is a description of a computation — not executed yet
// Run it:
Effect.runPromise(fetchUser('1')).then(console.log).catch(console.error);
```

---

## Typed Errors (No Thrown Exceptions)

```ts
class UserNotFound extends Data.TaggedError('UserNotFound')<{ id: string }> {}
class DatabaseError extends Data.TaggedError('DatabaseError')<{ message: string }> {}

// TypeScript KNOWS this can fail with UserNotFound | DatabaseError
const getUser = (id: string): Effect.Effect<User, UserNotFound | DatabaseError, never> =>
  pipe(
    Effect.tryPromise({
      try: () => db.findUser(id),
      catch: (err) => new DatabaseError({ message: String(err) }),
    }),
    Effect.flatMap((user) =>
      user ? Effect.succeed(user) : Effect.fail(new UserNotFound({ id }))
    )
  );

// Handle errors exhaustively
pipe(
  getUser('1'),
  Effect.catchTag('UserNotFound', (err) => Effect.succeed({ name: 'Anonymous' })),
  Effect.catchTag('DatabaseError', (err) => Effect.die(err)),  // unrecoverable
);
```

---

## Services and Layers (Dependency Injection)

```ts
import { Context, Layer, Effect } from 'effect';

// Define a service interface
class UserRepository extends Context.Tag('UserRepository')<
  UserRepository,
  {
    findById: (id: string) => Effect.Effect<User | null, DatabaseError>;
    create: (data: CreateUserData) => Effect.Effect<User, DatabaseError>;
  }
>() {}

// Use the service (requirement type includes UserRepository)
const getUser = (id: string) =>
  Effect.gen(function* () {
    const repo = yield* UserRepository;
    const user = yield* repo.findById(id);
    if (!user) return yield* Effect.fail(new UserNotFound({ id }));
    return user;
  });

// Implement the service
const UserRepositoryLive = Layer.succeed(UserRepository, {
  findById: (id) => Effect.tryPromise({
    try: () => db.users.findById(id),
    catch: (err) => new DatabaseError({ message: String(err) }),
  }),
  create: (data) => Effect.tryPromise({
    try: () => db.users.create(data),
    catch: (err) => new DatabaseError({ message: String(err) }),
  }),
});

// Provide the implementation when running
pipe(
  getUser('1'),
  Effect.provide(UserRepositoryLive),
  Effect.runPromise,
);
```

---

## Fibers — Structured Concurrency

```ts
import { Effect, Fiber } from 'effect';

// Run effects concurrently
const concurrent = Effect.gen(function* () {
  const [users, products] = yield* Effect.all(
    [fetchUsers(), fetchProducts()],
    { concurrency: 2 }
  );
  return { users, products };
});

// Race — first one wins
const fastest = yield* Effect.race(fetchFromPrimary(), fetchFromFallback());

// Fork into a background fiber
const fiber = yield* Effect.fork(longRunningOperation());
// ... do other work ...
const result = yield* Fiber.join(fiber);

// Interrupt a fiber
yield* Fiber.interrupt(fiber);
```

---

## Schema (Validation)

```ts
import { Schema } from '@effect/schema';

const UserSchema = Schema.Struct({
  id: Schema.String.pipe(Schema.pattern(/^[a-z0-9-]+$/)),
  email: Schema.String.pipe(Schema.filter(isEmail)),
  age: Schema.Number.pipe(Schema.between(0, 150)),
  role: Schema.Literal('admin', 'user', 'guest'),
});

type User = Schema.Schema.Type<typeof UserSchema>;

// Parse with typed errors
const parseUser = Schema.decode(UserSchema);

const result = yield* parseUser({ id: '1', email: 'alice@example.com', age: 25, role: 'user' });
// Error path: Effect.Effect<User, ParseError, never>
```

---

## Practical React Usage

```tsx
import { Effect, Layer } from 'effect';
import { useEffect, useState } from 'react';

function useEffect_<A, E>(
  effect: Effect.Effect<A, E, never>
): { data: A | null; error: E | null; loading: boolean } {
  const [state, setState] = useState({ data: null, error: null, loading: true });

  useEffect(() => {
    let cancelled = false;
    Effect.runPromise(effect)
      .then(data => { if (!cancelled) setState({ data, error: null, loading: false }); })
      .catch(error => { if (!cancelled) setState({ data: null, error, loading: false }); });
    return () => { cancelled = true; };
  }, []);

  return state;
}

function UserProfile({ id }: { id: string }) {
  const { data: user, loading, error } = useEffect_(
    pipe(
      getUser(id),
      Effect.provide(UserRepositoryLive)
    )
  );

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <div>{user!.name}</div>;
}
```

---

## Comparison with fp-ts

| | Effect-TS | fp-ts |
|---|---|---|
| Error handling | Built-in typed errors | Either monad |
| Async | Fibers, structured concurrency | TaskEither |
| Dependency injection | Layers/Context | Manual (Reader monad) |
| Schema validation | Built-in | Third-party |
| Observability | OpenTelemetry built-in | Manual |
| Bundle size | Larger | Smaller |
| Learning curve | Steeper | Also steep |
| Actively developed | Yes (very active) | Maintenance mode |

---

## Common Interview Questions

**Q: Why use Effect over async/await + try/catch?**
Typed errors: TypeScript knows exactly what errors can occur (no surprise `unknown`). Composable: chain and transform effects without nested try/catch. Testable: inject dependencies via Layers, no module mocks needed. Structured concurrency: fibers don't leak when cancelled.

**Q: What's the relationship between Effect and fp-ts?**
Effect-TS was heavily inspired by fp-ts and ZIO (Scala). The original author of fp-ts (Giulio Canti) contributed to Effect. Effect is considered fp-ts's spiritual successor with a more ergonomic API and richer features.

**Q: Is Effect too complex for typical frontend work?**
For simple CRUD apps, yes — the overhead isn't justified. For complex business logic with many error states, async operations, and service dependencies (large enterprise apps), Effect's type safety catches real bugs.
