# Utility Types in TypeScript

## The Idea

**In plain English:** Utility types are ready-made tools built into TypeScript that let you take an existing description of data and reshape it — for example, making every field optional, removing a field, or grabbing only a few fields — without rewriting the whole description from scratch.

**Real-world analogy:** Imagine a standard employment form with fields for name, address, phone, salary, and start date. When you need a quick contact card, you do not reprint the whole form — you just photocopy only the name and phone section. When someone is still interviewing, you hand them the same form but mark every field as "fill in later." TypeScript utility types do exactly this for your data shapes.

- The full employment form = the original TypeScript type (e.g., `User`)
- Photocopying only certain fields = `Pick<User, "name" | "phone">`
- Marking every field as "fill in later" = `Partial<User>`
- The finished, fully filled-out form = `Required<User>`

---

## Overview

TypeScript ships with a set of built-in generic type utilities that transform types — making properties optional or required, picking subsets of properties, constructing record types, and extracting type information from functions and classes. These utility types eliminate the need to hand-write common type transformations and are themselves implemented using mapped types and conditional types, so studying them is also a lesson in TypeScript's underlying type machinery.

## Property Modifier Utilities

### `Partial<T>`

Makes all properties of `T` optional:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "user";
}

type UserPatch = Partial<User>;
// { id?: number; name?: string; email?: string; role?: "admin" | "user" }

// Common use case: update/patch payloads
async function updateUser(id: number, patch: Partial<User>): Promise<User> {
  const existing = await fetchUser(id);
  return { ...existing, ...patch };
}

updateUser(1, { name: "Bob" }); // Only updating name — OK
```

Implementation:
```typescript
type Partial<T> = {
  [K in keyof T]?: T[K];
};
```

### `Required<T>`

Makes all properties of `T` required (removes optionality):

```typescript
interface Config {
  host?: string;
  port?: number;
  timeout?: number;
}

type RequiredConfig = Required<Config>;
// { host: string; port: number; timeout: number }

// Use case: after validation, assert all fields are present
function validateConfig(config: Config): RequiredConfig {
  if (!config.host || !config.port) throw new Error("Missing required fields");
  return config as RequiredConfig;
}
```

### `Readonly<T>`

Makes all properties of `T` readonly:

```typescript
interface State {
  count: number;
  items: string[];
}

type ImmutableState = Readonly<State>;

const state: ImmutableState = { count: 0, items: [] };
state.count = 1; // Error: cannot assign to 'count' because it is a read-only property

// Use case: Redux-style state, frozen configuration
function createInitialState(): Readonly<State> {
  return { count: 0, items: [] };
}
```

Note: `Readonly` is shallow. Nested objects are not automatically frozen:

```typescript
const state: Readonly<State> = { count: 0, items: [] };
state.items.push("new item"); // This works! items is readonly but the array itself is mutable
```

For deep immutability, use a recursive type or a library like `type-fest`'s `DeepReadonly`.

## Property Selection Utilities

### `Pick<T, K>`

Constructs a type with only the specified keys from `T`:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  passwordHash: string;
  createdAt: Date;
}

type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: number; name: string; email: string }

// Use case: safe DTOs that exclude sensitive fields
function toPublicUser(user: User): PublicUser {
  const { id, name, email } = user;
  return { id, name, email };
}
```

Implementation:
```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

### `Omit<T, K>`

Constructs a type by removing specified keys from `T`:

```typescript
type UserWithoutPassword = Omit<User, "passwordHash">;
// { id: number; name: string; email: string; createdAt: Date }

// More readable for removing a few fields from large types
type CreateUserPayload = Omit<User, "id" | "createdAt">;
// { name: string; email: string; passwordHash: string }
```

Implementation note: `Omit` is actually implemented as:
```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

The `K extends keyof any` (rather than `K extends keyof T`) is a deliberate design choice that allows `Omit` to accept keys not present in `T` without error.

### `Pick` vs `Omit`: Which to Use

Use `Pick` when you are explicitly selecting a small subset of properties — it documents what you need. Use `Omit` when you want everything except a few properties. As a rule of thumb: if the picked set is smaller than the omitted set, use `Pick`; otherwise, use `Omit`.

## Set Operation Utilities

### `Exclude<T, U>`

From a union type `T`, removes types that are assignable to `U`:

```typescript
type T = string | number | boolean | null | undefined;

type NonNullable_T = Exclude<T, null | undefined>;
// string | number | boolean

type OnlyPrimitives = Exclude<string | number | object | null, object | null>;
// string | number
```

### `Extract<T, U>`

The opposite of `Exclude` — keeps only types from `T` that are assignable to `U`:

```typescript
type T = string | number | boolean | (() => void);

type FunctionTypes = Extract<T, Function>;
// () => void

type StringOrNumber = Extract<T, string | number>;
// string | number
```

### `NonNullable<T>`

Removes `null` and `undefined` from `T`:

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

// Implementation:
type NonNullable<T> = T & {};
// (anything & {} removes null/undefined because null/undefined are not assignable to {})
```

## Record Type

### `Record<K, V>`

Constructs an object type with keys of type `K` and values of type `V`:

```typescript
type Role = "admin" | "user" | "guest";
type RolePermissions = Record<Role, string[]>;

const permissions: RolePermissions = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  guest: ["read"],
};

// Use case: lookup tables / maps
type CountryCode = "US" | "UK" | "CA";
type PhonePrefix = Record<CountryCode, string>;

const prefixes: PhonePrefix = {
  US: "+1",
  UK: "+44",
  CA: "+1",
};
```

Implementation:
```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

`Record<string, unknown>` is the type-safe alternative to `object` for dictionary-like structures.

## Function-Related Utilities

### `ReturnType<T>`

Extracts the return type of a function type `T`:

```typescript
function fetchUser(id: string) {
  return { id, name: "Alice", email: "alice@example.com" };
}

type FetchUserResult = ReturnType<typeof fetchUser>;
// { id: string; name: string; email: string }

// Useful when the return type is complex or inferred
async function loadData() {
  const response = await fetch("/api/data");
  return response.json() as Promise<{ items: string[] }>;
}

type LoadDataResult = Awaited<ReturnType<typeof loadData>>;
// { items: string[] }
```

Implementation:
```typescript
type ReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : any;
```

### `Parameters<T>`

Extracts the parameter types of a function as a tuple:

```typescript
function createUser(name: string, age: number, role: "admin" | "user") {
  return { name, age, role };
}

type CreateUserParams = Parameters<typeof createUser>;
// [name: string, age: number, role: "admin" | "user"]

// Use case: forwarding arguments to another function
function withLogging<T extends (...args: any[]) => any>(fn: T) {
  return (...args: Parameters<T>): ReturnType<T> => {
    console.log("calling with", args);
    return fn(...args);
  };
}
```

### `ConstructorParameters<T>`

Like `Parameters` but for class constructors:

```typescript
class Database {
  constructor(host: string, port: number, dbName: string) { }
}

type DBConstructorArgs = ConstructorParameters<typeof Database>;
// [host: string, port: number, dbName: string]
```

### `InstanceType<T>`

Extracts the instance type of a constructor function:

```typescript
class UserService {
  findById(id: string): Promise<User> { /* ... */ }
}

type UserServiceInstance = InstanceType<typeof UserService>;
// UserService (the instance type)

// Use case: factory functions that take a class and return an instance
function createService<T extends new (...args: any[]) => any>(
  Service: T
): InstanceType<T> {
  return new Service();
}
```

## String Manipulation Types

TypeScript 4.1 introduced intrinsic string manipulation types:

```typescript
type Greeting = "hello world";

type Upper = Uppercase<Greeting>;   // "HELLO WORLD"
type Lower = Lowercase<"FOO BAR">;  // "foo bar"
type Cap = Capitalize<"hello">;     // "Hello"
type Uncap = Uncapitalize<"Hello">; // "hello"
```

These are most useful when combined with template literal types:

```typescript
type EventName = "click" | "focus" | "blur";
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"
```

## Awaited Utility

### `Awaited<T>`

Recursively unwraps Promise types:

```typescript
type P1 = Awaited<Promise<string>>;              // string
type P2 = Awaited<Promise<Promise<number>>>;     // number
type P3 = Awaited<string | Promise<boolean>>;    // string | boolean
```

This replaced manual `infer` patterns for extracting resolved Promise values and is used internally by TypeScript to type `async` functions.

## Composing Utility Types

Utility types can be combined for precise transformations:

```typescript
// Make some properties required and others optional
type PartialExcept<T, K extends keyof T> = Partial<Omit<T, K>> & Required<Pick<T, K>>;

interface BlogPost {
  id: number;
  title: string;
  content?: string;
  publishedAt?: Date;
  authorId: number;
}

// id and authorId required, everything else optional
type CreateBlogPost = PartialExcept<BlogPost, "authorId" | "title">;
// { content?: string; publishedAt?: Date; id?: number; authorId: number; title: string }
```

```typescript
// Deep partial — recursively make all properties optional
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;
```

## Comparison Table

| Utility Type | Input | Output | Use Case |
|-------------|-------|--------|----------|
| `Partial<T>` | Object type | All props optional | Update/patch payloads |
| `Required<T>` | Object type | All props required | Post-validation assertions |
| `Readonly<T>` | Object type | All props readonly | Immutable state/config |
| `Pick<T, K>` | Object + keys | Subset of T | DTOs, safe projections |
| `Omit<T, K>` | Object + keys | T minus keys | Removing sensitive fields |
| `Exclude<T, U>` | Union type | T minus U members | Filtering union members |
| `Extract<T, U>` | Union type | T intersect U | Keeping union members |
| `NonNullable<T>` | Any type | T minus null/undefined | Asserting presence |
| `Record<K, V>` | Keys + value type | Mapped object | Lookup tables, maps |
| `ReturnType<T>` | Function type | Return type | Inferring from existing functions |
| `Parameters<T>` | Function type | Param tuple | Forwarding, wrapping |
| `Awaited<T>` | Promise/value | Resolved type | Async result types |

## Best Practices

### 1. Use `ReturnType` to Stay in Sync with Implementation

```typescript
// Instead of manually typing the return:
function buildConfig() {
  return { host: "localhost", port: 3000, debug: false };
}

type Config = ReturnType<typeof buildConfig>;
// Automatically stays in sync when buildConfig changes
```

### 2. Prefer `Omit` for Database Entities with Auto-Generated Fields

```typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: Date;
}

type CreateTodoInput = Omit<Todo, "id" | "createdAt">;
// { text: string; completed: boolean }
```

### 3. Use `Record` for Exhaustive Mappings

When keys come from a union, `Record` ensures all cases are handled:

```typescript
type Status = "pending" | "active" | "inactive" | "deleted";
type StatusLabel = Record<Status, string>;

// TypeScript errors if any Status value is missing
const labels: StatusLabel = {
  pending: "Pending Review",
  active: "Active",
  inactive: "Inactive",
  deleted: "Deleted",
};
```

### 4. Avoid Deep Nesting of Utility Types

Deep nesting makes error messages harder to read. Use intermediate type aliases:

```typescript
// Hard to read errors
type X = Readonly<Partial<Pick<Omit<User, "passwordHash">, "name" | "email">>>;

// Easier to debug
type PublicUser = Omit<User, "passwordHash">;
type PublicUserProfile = Pick<PublicUser, "name" | "email">;
type ReadonlyProfilePatch = Readonly<Partial<PublicUserProfile>>;
```

### 5. Use `Awaited` Instead of Manual Promise Unwrapping

```typescript
// Old pattern (manual infer)
type Unwrap<T> = T extends Promise<infer U> ? U : T;

// Modern — use the built-in
type ResolvedType = Awaited<SomePromise>;
```

## Interview Questions

### Q1: What is the difference between `Omit` and `Exclude`?

**Answer:** `Omit<T, K>` operates on object types and removes specified property keys, returning a new object type without those keys. `Exclude<T, U>` operates on union types and removes union members that are assignable to `U`. They solve different problems: `Omit` is for object shapes, `Exclude` is for union members. For example, `Omit<User, "password">` removes the password property from the User object type; `Exclude<"a" | "b" | "c", "a">` produces `"b" | "c"`.

### Q2: How is `Partial<T>` implemented internally?

**Answer:** `Partial<T>` is implemented as a mapped type: `type Partial<T> = { [K in keyof T]?: T[K]; }`. The `?` modifier makes each property optional. This iterates over all keys `K` in `T` and maps them to the same value type `T[K]` but with the optional modifier applied. Understanding this helps you write custom variants — for example, `DeepPartial` recursively applies the same transformation to nested objects.

### Q3: Why is `Record<string, unknown>` preferred over `object` for dictionaries?

**Answer:** `object` is the type of any non-primitive value in TypeScript. It does not tell you what properties the object has, and you cannot safely index it with a string key. `Record<string, unknown>` explicitly models a dictionary with string keys and values of `unknown` type, which you must narrow before use. This is safer than `Record<string, any>` (which skips type checks) and more expressive than `object`. Additionally, `Record<LiteralUnion, V>` ensures all keys from a known union are present.

### Q4: When would you use `Required<T>` in practice?

**Answer:** `Required<T>` is useful after a validation step to assert that a previously partial structure now has all fields. For example, an object built incrementally from user input might be typed as `Partial<Config>`. After validation confirms all required fields are present, you can cast to `Required<Config>` to unlock property access without null checks. It is also used in testing utilities to generate complete mock objects from optional-heavy interfaces.

### Q5: What is the difference between `Parameters<T>` and `ConstructorParameters<T>`?

**Answer:** Both extract parameter types as tuples, but from different contexts. `Parameters<T>` works on regular function types: `T extends (...args: any) => any`. `ConstructorParameters<T>` works on class constructors: `T extends abstract new (...args: any) => any`. Use `Parameters` for functions and methods; use `ConstructorParameters` when you need the arguments required to instantiate a class — for example, in factory functions or dependency injection containers.

## Common Pitfalls

### 1. `Partial` Does Not Make Nested Objects Partial

```typescript
interface Order {
  id: string;
  address: {
    street: string;
    city: string;
  };
}

type PartialOrder = Partial<Order>;
// address?: { street: string; city: string } — address is optional but NOT its contents

const patch: PartialOrder = {
  address: { street: "123 Main St" } // Error: missing 'city'
};
```

### 2. `Omit` Accepts Non-Existent Keys Without Error

```typescript
type User = { id: number; name: string };
type Bad = Omit<User, "email">; // No error — 'email' doesn't exist, but TypeScript allows it

// This can mask typos. Use this pattern to enforce key existence:
type StrictOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
type Fixed = StrictOmit<User, "email">; // Error: 'email' is not assignable to keyof User
```

### 3. `Record` With `string` Does Not Enforce Completeness

```typescript
// This is fine — keys are unconstrained
const map: Record<string, number> = { a: 1 };

// Only literal union keys enforce completeness:
type Color = "red" | "green" | "blue";
const colorMap: Record<Color, string> = {
  red: "#f00",
  green: "#0f0",
  // Error: missing 'blue'
};
```

### 4. `ReturnType` on Overloaded Functions Returns the Last Overload

```typescript
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}

type R = ReturnType<typeof format>; // string — last overload wins
// This is a TypeScript limitation; ReturnType does not union all overload return types
```

### 5. Confusing `Exclude` With `Omit`

```typescript
// Wrong: Exclude does not work on object types
type Bad = Exclude<User, "name">; // User (unchanged — "name" is not assignable to User)

// Correct: Omit removes object properties
type Good = Omit<User, "name">; // { id: number }

// Correct: Exclude removes from unions
type T = Exclude<"a" | "b" | "c", "a">; // "b" | "c"
```

## Resources

### Official Documentation
- [TypeScript Handbook: Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [TypeScript Source: Built-in Utilities](https://github.com/microsoft/TypeScript/blob/main/src/lib/es5.d.ts)

### Articles and Guides
- [Total TypeScript: Utility Types](https://www.totaltypescript.com/typescript-utility-types)
- [TypeScript Deep Dive: Type Compatibility](https://basarat.gitbook.io/typescript/)
- [Effective TypeScript: Items 14-18 on Type Operations](https://effectivetypescript.com/)

### Tools
- [TypeScript Playground](https://www.typescriptlang.org/play): Experiment with utility types
- [type-fest](https://github.com/sindresorhus/type-fest): Extended collection of utility types beyond the built-ins
- [ts-toolbelt](https://github.com/millsp/ts-toolbelt): Advanced utility types for complex transformations
