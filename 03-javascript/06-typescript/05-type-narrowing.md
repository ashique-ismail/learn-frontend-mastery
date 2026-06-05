# Type Narrowing, Type Guards, and Discriminated Unions

## The Idea

**In plain English:** Type narrowing is how TypeScript figures out, at a specific point in your code, that a value must be one specific type rather than several possibilities. Think of it as TypeScript "zooming in" on what a variable actually is after you've checked it with an if-statement or similar condition.

**Real-world analogy:** Imagine a package sorting facility where incoming boxes are labeled "fragile OR heavy" — the worker doesn't know which until they pick it up and feel the weight. Once they check ("this one's light, so it must be fragile"), they now handle it differently than a heavy box. TypeScript does the same thing automatically when it sees your checks.

- The box labeled "fragile OR heavy" = a variable with a union type like `string | number`
- The worker checking the weight = your `if (typeof value === "string")` condition
- How the worker handles the box after the check = the narrowed type TypeScript uses inside that `if` block

---

## Overview

Type narrowing is TypeScript's process of refining a type to a more specific type within a particular code path. When you check whether a value is a string before calling `.toUpperCase()`, TypeScript recognizes that check and narrows the type automatically. TypeScript's control flow analysis tracks narrowing across `if`, `switch`, `while`, `try/catch`, and early returns — making it one of the most practical features for safely working with `unknown`, `any`, and union types.

## Control Flow Based Narrowing

TypeScript's compiler performs control flow analysis to narrow types at each point in the code:

```typescript
function process(value: string | number): string {
  if (typeof value === "string") {
    // Here TypeScript knows value: string
    return value.toUpperCase();
  }
  // Here TypeScript knows value: number
  return value.toFixed(2);
}
```

TypeScript tracks narrowing across branches and at each statement:

```typescript
function describe(x: string | null | undefined): string {
  if (x == null) {
    // null == null and undefined == null both return true
    // TypeScript narrows: x is null | undefined here
    return "nothing";
  }
  // x is string here
  return x.trim();
}
```

## Built-in Narrowing Mechanisms

### `typeof` Guards

Narrows primitive types:

```typescript
function format(value: string | number | boolean): string {
  if (typeof value === "string") return value.trim();
  if (typeof value === "number") return value.toFixed(2);
  return String(value);
}
```

Valid `typeof` checks: `"string"`, `"number"`, `"bigint"`, `"boolean"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`.

Note: `typeof null === "object"` — this is a JavaScript quirk, not a narrowing of `null`.

### `instanceof` Guards

Narrows class instances:

```typescript
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

class ValidationError extends Error {
  constructor(public fields: string[], message: string) {
    super(message);
  }
}

function handleError(error: unknown): string {
  if (error instanceof ApiError) {
    return `API error ${error.statusCode}: ${error.message}`;
  }
  if (error instanceof ValidationError) {
    return `Validation failed: ${error.fields.join(", ")}`;
  }
  if (error instanceof Error) {
    return error.message;
  }
  return "Unknown error";
}
```

### Equality Narrowing

Strict equality (`===`) and loose equality (`==`) both narrow:

```typescript
function process(x: string | null | undefined): string {
  if (x === null) {
    return "null";
  }
  if (x === undefined) {
    return "undefined";
  }
  return x; // x: string
}

// Loose equality with null catches both null and undefined:
function trim(x: string | null | undefined): string {
  if (x == null) return "";
  return x.trim(); // x: string
}
```

### `in` Operator Narrowing

The `in` operator narrows based on property existence:

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };
type Duck = Fish & Bird;

function move(animal: Fish | Bird): void {
  if ("swim" in animal) {
    animal.swim(); // animal: Fish | Duck
  } else {
    animal.fly(); // animal: Bird
  }
}
```

### Truthiness Narrowing

Falsy values (`false`, `0`, `""`, `null`, `undefined`, `NaN`) are filtered by truthiness checks:

```typescript
function processInput(value: string | null | undefined): string {
  if (value) {
    // value: string (null, undefined, and "" are all filtered out)
    return value.trim();
  }
  return "";
}
```

Be careful: truthiness narrowing removes `""` (empty string) along with `null` and `undefined`, which may not be intended.

## User-Defined Type Guards

When TypeScript's built-in narrowing is insufficient, you can define custom type predicates:

### Type Predicate Functions

```typescript
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value &&
    typeof (value as any).id === "number" &&
    typeof (value as any).name === "string"
  );
}

// Usage
function processUser(data: unknown): string {
  if (isUser(data)) {
    return data.name; // TypeScript knows data: User
  }
  throw new Error("Invalid user data");
}
```

The `value is T` return type is the type predicate. It tells TypeScript: "if this function returns `true`, narrow the parameter to type `T` in the calling scope."

### Array Type Guards

```typescript
function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

const items: (string | null | undefined)[] = ["a", null, "b", undefined, "c"];
const filtered: string[] = items.filter(isNonNullable);
// Without the type guard, .filter would return (string | null | undefined)[]
```

### Assertion Functions

TypeScript 3.7 introduced assertion functions — they never return if the condition fails (they throw), and TypeScript narrows based on that:

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}

function assertDefined<T>(value: T): asserts value is NonNullable<T> {
  if (value === null || value === undefined) {
    throw new Error("Expected value to be defined");
  }
}

function processConfig(config: unknown): void {
  assertIsString(config); // Throws if not a string
  config.trim(); // TypeScript now knows config: string
}
```

Assertion functions narrow for the rest of the current scope, not just inside an `if` block.

## Discriminated Unions

A discriminated union (tagged union, algebraic data type) is a union of types sharing a common literal property — the discriminant. TypeScript uses the discriminant for exhaustive narrowing:

```typescript
type LoadingState = {
  status: "loading";
};

type SuccessState = {
  status: "success";
  data: User[];
  timestamp: Date;
};

type ErrorState = {
  status: "error";
  error: Error;
  retryable: boolean;
};

type RequestState = LoadingState | SuccessState | ErrorState;
```

### Exhaustive Switch Statements

```typescript
function render(state: RequestState): string {
  switch (state.status) {
    case "loading":
      return "Loading...";
    case "success":
      return `Loaded ${state.data.length} users`;
    case "error":
      return `Error: ${state.error.message}`;
    default:
      // TypeScript narrows to `never` here — if you add a new state, this errors
      const _exhaustive: never = state;
      throw new Error(`Unhandled status: ${(_exhaustive as any).status}`);
  }
}
```

### The Exhaustiveness Check Pattern

The `never` assignment in the `default` case is a compile-time check. If you add a new variant to `RequestState` without updating `render`, TypeScript will error because the new variant is not `never`:

```typescript
// If you add: type CancelledState = { status: "cancelled" }
// and update RequestState to include it, TypeScript errors:
// "Type 'CancelledState' is not assignable to type 'never'"
```

### Nested Discriminated Unions

```typescript
type TextContent = {
  type: "text";
  text: string;
};

type ImageContent = {
  type: "image";
  src: string;
  alt: string;
};

type VideoContent = {
  type: "video";
  src: string;
  duration: number;
  subtitles?: string;
};

type Content = TextContent | ImageContent | VideoContent;

function renderContent(content: Content): HTMLElement {
  switch (content.type) {
    case "text": {
      const el = document.createElement("p");
      el.textContent = content.text;
      return el;
    }
    case "image": {
      const el = document.createElement("img");
      el.src = content.src;
      el.alt = content.alt;
      return el;
    }
    case "video": {
      const el = document.createElement("video");
      el.src = content.src;
      return el;
    }
  }
}
```

## Narrowing with Assignment

TypeScript narrows based on assigned values:

```typescript
let value: string | number;
value = "hello";
// value: string (TypeScript tracks the assignment)
console.log(value.toUpperCase()); // OK

value = 42;
// value: number now
console.log(value.toFixed(2)); // OK
```

## Narrowing Through Early Returns

TypeScript tracks control flow through early returns, eliminating branches:

```typescript
function getUser(id: string | null): User {
  if (!id) {
    throw new Error("ID is required");
  }
  // id: string (the null case was thrown above)
  return fetchUser(id);
}
```

## The `satisfies` Operator (TypeScript 4.9)

`satisfies` checks that a value matches a type without widening:

```typescript
type ColorMap = Record<string, [number, number, number] | string>;

// Without satisfies: all values are typed as [number, number, number] | string
const palette: ColorMap = {
  red: [255, 0, 0],
  green: "#00ff00",
};

palette.red.toUpperCase(); // Error — TypeScript sees [number, number, number] | string

// With satisfies: checks the type, but infers the narrowest type for each value
const palette2 = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies ColorMap;

palette2.red[0]; // number — TypeScript knows it's a tuple
palette2.green.toUpperCase(); // string — TypeScript knows it's a string
```

## Comparison Table

| Mechanism | Type Narrowed | When to Use |
|-----------|--------------|-------------|
| `typeof` | Primitives | `string`, `number`, `boolean`, `undefined` |
| `instanceof` | Class instances | Custom classes, built-ins (`Date`, `Error`, `Map`) |
| `in` | Property existence | Structural narrowing, duck typing |
| Equality (`===`) | Exact value | Discriminants, null checks |
| Truthiness | Falsy values | Quick null/undefined guards (mind `""`) |
| Type predicate | Any type | Complex runtime validation |
| Assertion function | Any type | Throw-on-failure guards (post-call narrowing) |
| Discriminated union | Tagged union members | Exhaustive state machines |
| `satisfies` | Expression type | Check type without widening |

## Best Practices

### 1. Model State with Discriminated Unions, Not Optional Properties

```typescript
// Fragile
interface State {
  loading?: boolean;
  data?: User;
  error?: string;
}

// Valid but ambiguous: { loading: true, data: user } — what is this?

// Clear and exhaustive
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: string };
```

### 2. Always Add Exhaustiveness Checks

```typescript
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

// Reusable in any switch
switch (shape.kind) {
  case "circle": ...
  case "rectangle": ...
  default: return assertNever(shape);
}
```

### 3. Use Type Predicates for Runtime Parsing

```typescript
function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

function parseApiResponse(raw: unknown): ApiResponse {
  if (!isRecord(raw)) throw new Error("Expected object");
  if (typeof raw.status !== "number") throw new Error("Expected status");
  // ...
}
```

### 4. Avoid Widening with `as` When a Guard Would Work

```typescript
// Unsafe: bypasses type checking
const user = response.data as User;

// Safe: validates at runtime
if (isUser(response.data)) {
  const user = response.data; // User
}
```

### 5. Combine Assertion Functions with Validation Libraries

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
});

function assertUser(value: unknown): asserts value is z.infer<typeof UserSchema> {
  UserSchema.parse(value); // throws on failure
}
```

## Interview Questions

### Q1: What is the difference between a type predicate and an assertion function?

**Answer:** A type predicate (`value is T`) is used on functions that return `boolean`. If the function returns `true`, TypeScript narrows the argument to `T` in the calling branch. An assertion function (`asserts value is T`) is used on functions that either return normally (narrowing the type for the rest of the scope) or throw. Assertion functions narrow unconditionally after the call, rather than within an `if` branch. Use type predicates for conditional narrowing (`.filter`, `if` guards); use assertion functions when a failure should throw and the happy path should be narrowed.

### Q2: Why is a discriminated union safer than optional properties for state modeling?

**Answer:** Optional properties create combinatorial ambiguity — an object with `{ loading?: boolean; data?: User; error?: string }` allows impossible states like `{ loading: true, data: user }`. The compiler cannot tell which combination is valid, so it cannot help you handle each case correctly. A discriminated union with a `status` literal field makes states mutually exclusive and exhaustive. TypeScript narrows each `case` in a `switch` statement, gives you only the properties valid for that state, and errors at compile time if you forget to handle a case (via the `never` exhaustiveness check).

### Q3: What is control flow analysis and how does TypeScript use it?

**Answer:** Control flow analysis is TypeScript's tracking of type narrowing through the code graph. At each point in the code, TypeScript computes the set of types a variable can have based on all prior checks and assignments. It understands `if`, `else`, `switch`, `while`, `for`, `try/catch`, early `return`, and `throw` statements. When you check `if (x !== null)`, TypeScript removes `null` from `x`'s type in the `true` branch. When you `throw` in a branch, TypeScript knows subsequent code is unreachable from that path. This analysis runs without any explicit annotations — it infers narrowing from standard JavaScript patterns.

### Q4: What is `satisfies` and when should you use it instead of a type annotation?

**Answer:** `satisfies` validates that an expression is compatible with a type, but infers the narrowest possible type for the expression itself. A direct type annotation widens the type — `const x: Foo = ...` makes `x` typed as `Foo`, losing specific literal types. `const x = ... satisfies Foo` checks compatibility with `Foo` at compile time but preserves the inferred type. Use `satisfies` when you want type-checking validation without losing narrowness — for example, palette/config objects where you want both type safety and access to the specific value types of each property.

### Q5: How does TypeScript narrow through assignment?

**Answer:** TypeScript tracks assignments and narrows the type of a variable based on the most recently assigned value. If you declare `let x: string | number` and then assign `x = "hello"`, TypeScript narrows `x` to `string` at all subsequent uses until another assignment. This is called "assignment narrowing." It works across reassignments and survives branching in some cases (when TypeScript can determine which branch was taken). This means you can often avoid explicit type assertions by structuring assignments correctly.

## Common Pitfalls

### 1. Truthiness Guards Remove Empty Strings

```typescript
function process(value: string | null): string {
  if (value) {
    return value.trim(); // "" never reaches here
  }
  return "default";
}

// Better for null-only check:
if (value !== null) { ... }
```

### 2. `typeof null === "object"` Does Not Narrow null

```typescript
function isObject(value: unknown): boolean {
  return typeof value === "object"; // WRONG: includes null
}

// Correct:
function isObject(value: unknown): value is object {
  return typeof value === "object" && value !== null;
}
```

### 3. Type Predicates Are Trusted Unconditionally

```typescript
// TypeScript trusts this even if the implementation is wrong
function isUser(value: unknown): value is User {
  return true; // TypeScript narrows to User even for non-user values!
}
```

Type predicates are a promise to the compiler — if the implementation is incorrect, you get runtime errors with no compile-time warning.

### 4. Narrowing Does Not Persist Through Non-Trivial Functions

```typescript
const values: (string | null)[] = ["a", null, "b"];

values.forEach(v => {
  if (v !== null) {
    // v: string inside this callback
  }
});

// But reassigning to a variable loses the narrowing:
let current: string | null = "hello";
// ... some time later ...
// TypeScript forgets the narrowing if there's a potential for reassignment
```

### 5. Discriminated Unions Require Exact Discriminant Literal Matches

```typescript
type A = { kind: "a"; value: string };
type B = { kind: "b"; value: number };
type AB = A | B;

function process(x: AB) {
  if (x.kind === "A") { // Capital A — no match, no narrowing!
    x.value; // still string | number
  }
}
```

## Resources

### Official Documentation
- [TypeScript Handbook: Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TypeScript Handbook: Type Guards and Differentiating Types](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)
- [TypeScript 3.7: Assertion Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions)
- [TypeScript 4.9: The `satisfies` Operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html)

### Articles and Guides
- [Total TypeScript: Type Guards](https://www.totaltypescript.com/type-guards-in-typescript)
- [TypeScript Deep Dive: Type Guards](https://basarat.gitbook.io/typescript/type-system/typeguard)
- [Narrowing and Discriminated Unions in Practice](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions)

### Tools
- [TypeScript Playground](https://www.typescriptlang.org/play): Hover over variables to see narrowed types
- [zod](https://github.com/colinhacks/zod): Runtime validation library with first-class TypeScript type inference
