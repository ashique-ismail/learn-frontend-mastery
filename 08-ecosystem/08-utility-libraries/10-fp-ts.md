# fp-ts

## The Idea

**In plain English:** fp-ts is a TypeScript library that gives you special "containers" to safely handle situations where a value might be missing or an operation might fail — instead of your program crashing unexpectedly, you plan for both outcomes up front. A container is just a wrapper around a value that carries extra information about whether the value is there or not.

**Real-world analogy:** Imagine ordering a package online. The delivery service gives you a tracking box: it either shows "delivered with your item inside" or "delivery failed with a reason why." You deal with both cases before you even open the door.

- The tracking box = the `Either` or `Option` container in fp-ts
- "Delivered with your item" = `Right` (success) or `Some` (value present)
- "Delivery failed with a reason" = `Left` (error) or `None` (value absent)

---

## What It Is

fp-ts brings functional programming patterns to TypeScript — type-safe representations for missing values (Option), errors (Either), async operations (Task/TaskEither), and composable functions (pipe/flow).

It's a library for developers who want the safety guarantees of functional programming without switching languages.

---

## `pipe` and `flow`

The foundation of fp-ts composition:

```ts
import { pipe, flow } from 'fp-ts/function';

// pipe: apply functions left to right to a value
const result = pipe(
  [1, 2, 3, 4, 5],
  xs => xs.filter(x => x % 2 === 0),  // [2, 4]
  xs => xs.map(x => x * 10),           // [20, 40]
  xs => xs.reduce((a, b) => a + b, 0), // 60
);

// flow: compose functions without an initial value
const transform = flow(
  (xs: number[]) => xs.filter(x => x % 2 === 0),
  xs => xs.map(x => x * 10),
  xs => xs.reduce((a, b) => a + b, 0),
);
transform([1, 2, 3, 4, 5]); // 60
```

---

## `Option<A>` — Type-Safe Nullability

Replaces `T | null | undefined` with an explicit type:

```ts
import * as O from 'fp-ts/Option';
import { pipe } from 'fp-ts/function';

// none: represents absence
// some(value): represents presence
const user: O.Option<User> = O.some({ id: '1', name: 'Alice' });
const noUser: O.Option<User> = O.none;

// Create
const maybeUser = O.fromNullable(possiblyNullUser); // null/undefined → none
const maybeAge = O.fromPredicate((n: number) => n > 0)(age); // -1 → none

// Transform (does nothing on none)
const maybeName = pipe(
  maybeUser,
  O.map(user => user.name),           // Option<string>
  O.map(name => name.toUpperCase()),  // Option<string>
);

// Extract (provide fallback for none)
const name = pipe(
  maybeUser,
  O.fold(
    () => 'Anonymous',          // none case
    user => user.name           // some case
  )
);
// or
const name2 = pipe(maybeName, O.getOrElse(() => 'Anonymous'));

// Chain (flatMap)
const address = pipe(
  maybeUser,
  O.chain(user => O.fromNullable(user.address)), // Option<Address>
);
```

---

## `Either<E, A>` — Type-Safe Error Handling

Represents either a failure (Left) or success (Right):

```ts
import * as E from 'fp-ts/Either';

// right: success value
// left: error value
type ValidationError = { field: string; message: string };

function parseAge(input: string): E.Either<ValidationError, number> {
  const age = parseInt(input, 10);
  if (isNaN(age)) return E.left({ field: 'age', message: 'Must be a number' });
  if (age < 0 || age > 150) return E.left({ field: 'age', message: 'Must be 0-150' });
  return E.right(age);
}

const result = pipe(
  parseAge('25'),
  E.map(age => age + 1),                      // transform success value
  E.mapLeft(err => ({ ...err, code: 400 })),  // transform error value
  E.fold(
    err => console.error(err.message),
    age => console.log(`Validated age: ${age}`),
  )
);
```

---

## `Task<A>` and `TaskEither<E, A>` — Async Operations

```ts
import * as TE from 'fp-ts/TaskEither';
import * as T from 'fp-ts/Task';

// TaskEither<E, A> = () => Promise<Either<E, A>>

function fetchUser(id: string): TE.TaskEither<Error, User> {
  return TE.tryCatch(
    () => api.getUser(id),          // async operation
    (error) => error as Error       // convert thrown value to left
  );
}

// Chain async operations
const program = pipe(
  fetchUser('1'),
  TE.chain(user =>
    TE.tryCatch(
      () => api.getPosts(user.id),
      err => err as Error
    )
  ),
  TE.map(posts => posts.filter(p => p.published)),
  TE.fold(
    err => T.of(`Error: ${err.message}`),
    posts => T.of(`Found ${posts.length} posts`),
  )
);

// Execute (it's just a function returning a Promise)
program().then(console.log);
```

---

## Practical React/Angular Usage

```ts
// API layer using TaskEither
function createUserApi(http: HttpClient) {
  return {
    getUser: (id: string): TE.TaskEither<Error, User> =>
      TE.tryCatch(
        () => lastValueFrom(http.get<User>(`/api/users/${id}`)),
        err => err as Error
      ),
  };
}

// React hook
function useUser(id: string) {
  const [state, setState] = useState<E.Either<Error, User> | null>(null);

  useEffect(() => {
    pipe(
      userApi.getUser(id),
      TE.match(
        err => setState(E.left(err)),
        user => setState(E.right(user))
      )
    )(); // execute the task
  }, [id]);

  return state;
}
```

---

## When Is fp-ts Worth It?

**Good fit:**
- Complex data transformation pipelines
- Applications where error handling must be exhaustive
- Teams comfortable with functional programming
- TypeScript-heavy backend services

**Not a fit:**
- Simple CRUD apps
- Teams unfamiliar with functional programming concepts
- When Effect-TS (fp-ts's spiritual successor) would be more appropriate

---

## Common Interview Questions

**Q: What's the difference between `map` and `chain` on Option/Either?**
`map` transforms the inner value with a function that returns a plain value. `chain` (flatMap) transforms with a function that returns the same container type — it "flattens" to avoid nested `Option<Option<A>>`.

**Q: How does fp-ts compare to Effect-TS?**
Effect-TS is fp-ts's spiritual successor with a richer API: dependency injection, resource management, structured concurrency, schema validation. fp-ts focuses on core algebraic abstractions (Functor, Applicative, Monad). For new projects, Effect-TS is often the better choice.

**Q: Is fp-ts suitable for frontend development?**
For complex business logic, yes. For simple UI state, it adds overhead. Many teams use fp-ts for the API/service layer (where error handling is critical) but plain TypeScript for component logic.
