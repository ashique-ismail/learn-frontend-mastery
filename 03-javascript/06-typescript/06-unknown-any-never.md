# `unknown` vs `any` vs `never` in TypeScript

## The Idea

**In plain English:** TypeScript has three special labels for values that are tricky to categorize: `any` means "I don't care, skip all checks," `unknown` means "I'm not sure yet, so force me to check before using it," and `never` means "this value literally cannot exist." Each label tells the compiler how strictly to guard your code.

**Real-world analogy:** Imagine a school lost-and-found box. Someone drops off a mystery bag and the front-desk staff has three ways to handle it:

- The "any" clerk hands the bag straight to anyone who asks, no questions asked — they just trust it is fine (no safety check).
- The "unknown" clerk keeps the bag behind the counter until you open it and confirm what is inside before they let you use it (forces a check first).
- The "never" clerk has a slot labelled "items that cannot exist here" — nothing can actually land in it, and if somehow something does, the whole system flags an error.

Explicit mappings:

- The mystery bag = a value whose type TypeScript does not know
- The "any" clerk = the `any` type (skips all type checking)
- The "unknown" clerk = the `unknown` type (allows any value in, but requires narrowing before use)
- The "never" slot = the `never` type (represents an impossible value or unreachable code path)

---

## Overview

Three special types in TypeScript occupy extreme positions in the type hierarchy: `any` is the escape hatch that disables type checking, `unknown` is the type-safe alternative for values whose type is truly unknown at compile time, and `never` is the bottom type representing values that can never exist. Understanding the distinctions between these types is fundamental to writing safe TypeScript and reasoning about the type system's structure.

## The Type Hierarchy

TypeScript's type system forms a lattice. At the top is `unknown` (every type is assignable to `unknown`). At the bottom is `never` (never is assignable to every type). `any` is special — it simultaneously acts as the top and bottom type, bypassing the normal hierarchy entirely.

```
         unknown (top type)
        /        |        \
   string     number    boolean  ...
        \        |        /
              never (bottom type)

any — outside the hierarchy, assignable to/from everything
```

## `any`: The Escape Hatch

### What `any` Does

`any` tells TypeScript: "stop checking this value." A value typed `any` can be assigned to any type and accepts any operation:

```typescript
let x: any = "hello";
x.toUpperCase();  // OK
x.foo.bar.baz;    // OK (no error — TypeScript stops checking)
x = 42;           // OK
x = null;         // OK
x = {};           // OK

const str: string = x; // OK — any is assignable to string
const num: number = x; // OK — any is assignable to number
```

### How `any` Spreads

`any` is contagious. Operations on `any` produce `any`:

```typescript
let data: any = fetchData();
const result = data.users[0].name; // result: any
const upper = result.toUpperCase(); // upper: any
// The `any` type spreads through the chain
```

This is why `any` can silently corrupt type safety across an entire codebase.

### When `any` Is Justified

- Gradual migration from JavaScript to TypeScript (temporary)
- Third-party libraries with no type definitions and no community-maintained `@types`
- Type system limitations where the correct type is provably correct but unrepresentable
- Test mocks where you need to partially implement an interface

```typescript
// Acceptable use: temp workaround during migration
const legacyModule = require("./old-module") as any;

// Unacceptable: using any to avoid thinking about types
function processData(data: any): any { ... }
```

### `noImplicitAny`

With `noImplicitAny: true` in `tsconfig.json` (default in `strict` mode), TypeScript errors when it cannot infer a type and would fall back to `any`:

```typescript
function greet(name) { // Error: Parameter 'name' implicitly has an 'any' type
  return `Hello, ${name}`;
}

// Fix: add annotation
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

## `unknown`: The Type-Safe Top Type

### What `unknown` Does

`unknown` is the type-safe counterpart to `any`. A value of type `unknown` can hold any value — but you cannot perform operations on it without first narrowing its type:

```typescript
let x: unknown = "hello";

x.toUpperCase(); // Error: Object is of type 'unknown'
x.length;        // Error

// Must narrow first:
if (typeof x === "string") {
  x.toUpperCase(); // OK — x: string in this branch
}
```

### Assignability of `unknown`

```typescript
// Any value is assignable TO unknown
let u: unknown;
u = "hello";
u = 42;
u = null;
u = undefined;
u = { name: "Alice" };
u = [1, 2, 3];

// unknown is assignable only to unknown and any
const str: string = u;   // Error
const num: number = u;   // Error
const a: any = u;        // OK (any accepts anything)
const u2: unknown = u;   // OK (unknown accepts unknown)
```

### Practical Use: Catch Clauses

Before TypeScript 4.0, `catch` clause variables were typed as `any`. Since 4.0, with `useUnknownInCatchVariables: true` (enabled by `strict`), they are `unknown`:

```typescript
try {
  await riskyOperation();
} catch (error) {
  // error: unknown (in strict mode)

  if (error instanceof Error) {
    console.error(error.message); // safe
  } else if (typeof error === "string") {
    console.error(error);
  } else {
    console.error("Unknown error", error);
  }
}
```

### Practical Use: Parsing External Data

```typescript
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();

  if (!isUser(data)) {
    throw new Error("Unexpected API response shape");
  }

  return data; // User — safely narrowed
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as any).id === "string" &&
    typeof (value as any).name === "string"
  );
}
```

### `unknown` in Generic Constraints

```typescript
// Before: use any for "I don't know what this is"
function logValue(value: any): void { console.log(value); }

// Better: use unknown to document intent without spreading any
function logValue(value: unknown): void { console.log(value); }

// Or with generics for relational typing
function identity<T>(value: T): T { return value; }
```

## `never`: The Bottom Type

### What `never` Represents

`never` is the type of values that can never exist. A variable of type `never` can hold no value — not even `null` or `undefined`. A function returning `never` either throws unconditionally or runs forever.

```typescript
function throwError(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {}
}

// never is assignable to every type
function coerce(): string {
  return throwError("this never returns"); // valid: never is assignable to string
}
```

### `never` in Union and Intersection Types

`never` is the identity element for unions and the absorbing element for intersections:

```typescript
type T1 = string | never;   // string (never disappears from unions)
type T2 = string & never;   // never (never absorbs intersections)
type T3 = never | never;    // never
```

### Exhaustiveness Checking

The primary practical use of `never` is exhaustiveness checking in switch statements:

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      // TypeScript narrows shape to `never` here if all cases are handled
      // If you add a new Shape variant and forget a case, this errors:
      // "Type 'NewShape' is not assignable to type 'never'"
      const _check: never = shape;
      throw new Error(`Unhandled shape kind: ${(_check as any).kind}`);
  }
}
```

### `never` in Conditional Types

Conditional types that return `never` effectively filter union members:

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
// Distributes over union, removing null and undefined

type FilterStrings<T> = T extends string ? T : never;
type StringsOnly = FilterStrings<string | number | boolean | null>;
// string (number, boolean, null filtered out via never)
```

### `never` in Impossible States

`never` can explicitly mark code paths that should be unreachable:

```typescript
function processAction(action: { type: "increment" } | { type: "decrement" }) {
  switch (action.type) {
    case "increment":
      return 1;
    case "decrement":
      return -1;
    default:
      // This is reached only if action.type is somehow a different string
      // TypeScript knows this is unreachable and narrows to never
      const _: never = action;
      return 0;
  }
}
```

### Branding with `never`

`never` appears in branded/opaque types to make them unsatisfiable without a cast:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

// Cannot accidentally mix IDs:
function getUser(id: UserId): User { ... }

const orderId: OrderId = "order-123" as OrderId;
getUser(orderId); // Error: OrderId is not assignable to UserId
```

## Comparison Table

| Aspect | `any` | `unknown` | `never` |
|--------|-------|-----------|---------|
| Position in hierarchy | Outside (top + bottom) | Top type | Bottom type |
| Assignable from | Everything | Everything | Nothing (only `never`) |
| Assignable to | Everything | Only `unknown` and `any` | Everything |
| Allows property access | Yes (no checks) | No (must narrow first) | N/A (no value exists) |
| Allows operations | Yes (no checks) | No (must narrow first) | N/A |
| Spreads through usage | Yes (operations produce `any`) | No (operations produce errors) | N/A |
| Main use case | Escape hatch, migration | Safe external data | Exhaustiveness, unreachable code |
| In union | Becomes `any` | Neutral (`string \| unknown = unknown`) | Disappears (`string \| never = string`) |
| In intersection | Becomes `any` | Neutral (`string & unknown = string`) | Absorbs (`string & never = never`) |

## Best Practices

### 1. Replace `any` with `unknown` for External Data

```typescript
// Bad: loses all type safety
function parse(data: any): any { return data; }

// Good: forces callers to narrow before use
function parse(data: unknown): unknown { return data; }

// Best: validate and return a typed value
function parse(data: unknown): ApiResponse {
  if (!isApiResponse(data)) throw new Error("Invalid response");
  return data;
}
```

### 2. Use `never` to Detect Unhandled Cases at Compile Time

```typescript
function assertNever(value: never, message = "Unhandled case"): never {
  throw new Error(`${message}: ${JSON.stringify(value)}`);
}

// Place in default case or after exhaustive if-else chains
```

### 3. Avoid Returning `any` from Functions

```typescript
// Leaks any to callers
function getConfig(): any { return config; }

// Forces callers to narrow unnecessarily
function getConfig(): unknown { return config; }

// Best: return the actual type
function getConfig(): AppConfig { return config; }
```

### 4. Prefer `unknown` over `{}` and `object` for "any value"

```typescript
// object — excludes primitives
function log(value: object): void { console.log(value); }
log(42); // Error

// {} — any non-nullish value (surprising behavior)
function log(value: {}): void { console.log(value); }
log(null); // Error

// unknown — truly any value including null/undefined
function log(value: unknown): void { console.log(value); }
log(null); // OK
```

### 5. Set `strict: true` to Catch Implicit `any`

```json
{
  "compilerOptions": {
    "strict": true
    // Enables: noImplicitAny, useUnknownInCatchVariables, strictNullChecks, etc.
  }
}
```

## Interview Questions

### Q1: Why is `unknown` safer than `any`?

**Answer:** With `any`, TypeScript completely disables type checking. Any operation on an `any` value is allowed, and the result is also `any`, spreading throughout the codebase. With `unknown`, the value can hold anything, but you must narrow it before performing any operations. This forces explicit validation at the point of use. `any` is a promise to TypeScript that you know what you are doing; `unknown` is an honest admission that you do not know the type yet, with the compiler enforcing that you figure it out before using the value.

### Q2: What does it mean that `never` is the bottom type?

**Answer:** In a type hierarchy, a bottom type is one that is a subtype of every other type. Being a subtype means it is assignable to every type — you can use a `never` value anywhere. This is logically consistent because a `never` value cannot exist, so there are no actual values to check. Functions that return `never` either throw unconditionally or loop forever; since they produce no value, returning `never` is vacuously valid for any return type context. In practice, `never` represents impossible code paths and is used to enforce exhaustiveness at compile time.

### Q3: What happens when `any` is in a union or intersection?

**Answer:** `any` in a union with other types produces `any` — it dominates. `string | any` is `any`. Similarly, `string & any` is `any`. This is the contagious behavior that makes `any` dangerous: it absorbs all type-level information. By contrast, `unknown` in a union absorbs the other members (`string | unknown = unknown`), and `never` in a union disappears (`string | never = string`), reflecting their proper positions in the type hierarchy.

### Q4: When should you use `never` explicitly vs. letting TypeScript infer it?

**Answer:** TypeScript infers `never` in unreachable branches and impossible type intersections automatically. You should use `never` explicitly when you want a compile-time guarantee — primarily in exhaustiveness checks (assigning to a `never`-typed variable in a `default` case forces TypeScript to error if new cases are added), in conditional types to filter union members, and in branded type patterns to mark property types that should never be satisfied. Using `never` explicitly documents intent and creates enforced invariants rather than relying on inference.

### Q5: How does `useUnknownInCatchVariables` change error handling?

**Answer:** Before TypeScript 4.0, `catch (error)` gave `error` the type `any`, allowing unsafe operations like `error.message` without any check. `useUnknownInCatchVariables` (enabled by `strict: true` in 4.0+) types `error` as `unknown` instead. This forces you to narrow the error type before accessing any properties. The idiomatic pattern is to check `error instanceof Error` first, then access `.message`. This is safer because `throw` can be called with any value — a string, a number, a custom object — not just `Error` instances.

## Common Pitfalls

### 1. Assuming `any` Is a "Strict" Unknown

```typescript
// This looks like a type guard, but the return type is `any`, not narrowed
function getData(): any {
  return fetch("/api").then(r => r.json());
}

const data = getData();
data.items.map(...); // No error, but crashes at runtime if items is absent
```

### 2. Using `as any` to Suppress Errors Rather Than Fix Them

```typescript
// Hiding the problem
const user = (response as any).data.user;

// Addressing the problem
if (isUser((response as ApiResponse).data.user)) {
  const user = (response as ApiResponse).data.user;
}
```

### 3. Thinking `never` Variables Can Be Used

```typescript
function process(x: never): void {
  console.log(x); // x: never — this runs if someone passes a value TypeScript thought was never
  // At runtime, this can still execute if TypeScript was wrong (e.g., via any)
}
```

`never` is a type-level concept. At runtime, values can still reach `never`-typed code paths if `any` was used upstream.

### 4. Returning `never` from Functions That Might Return

```typescript
function parseId(value: string): number | never {
  // never in a return type union is meaningless — it reduces to just number
  const n = Number(value);
  if (isNaN(n)) throw new Error("Invalid ID");
  return n;
}

// `number | never` is just `number`
// Use `never` only for functions that ALWAYS throw/loop
```

### 5. Confusing `void` and `never`

```typescript
// void — function returns but produces no useful value (may return undefined)
function log(msg: string): void {
  console.log(msg);
}

// never — function does not return at all
function crash(msg: string): never {
  throw new Error(msg);
}

// void is assignable to undefined; never is not assignable to void
const v: void = undefined; // OK
const n: never = undefined as never; // Not really OK — never has no values
```

## Resources

### Official Documentation
- [TypeScript Handbook: The `unknown` Type](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown)
- [TypeScript Handbook: `never`](https://www.typescriptlang.org/docs/handbook/2/functions.html#never)
- [TypeScript 3.0: `unknown` Type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#new-unknown-top-type)
- [TypeScript Handbook: Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)

### Articles and Guides
- [TypeScript: `unknown` vs `any`](https://mariusschulz.com/blog/the-unknown-type-in-typescript)
- [TypeScript Deep Dive: `never`](https://basarat.gitbook.io/typescript/type-system/never)
- [Effective TypeScript: Items 38-42 on `any`](https://effectivetypescript.com/)

### Tools
- [TypeScript Playground](https://www.typescriptlang.org/play): Experiment with assignability between `any`, `unknown`, and `never`
- [ESLint `@typescript-eslint/no-explicit-any`](https://typescript-eslint.io/rules/no-explicit-any/): Enforce avoidance of `any`
