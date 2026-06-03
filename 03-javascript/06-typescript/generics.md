# Generics in TypeScript

## Overview

Generics are TypeScript's mechanism for writing reusable, type-safe code that works across multiple types without sacrificing type information. Rather than accepting `any` and losing all type checks, generics allow you to parameterize types — deferring the decision of what type to use until the call site. Generics appear in functions, classes, interfaces, and type aliases, and they underpin most of TypeScript's standard library.

## Generic Functions

### Basic Syntax

A generic function declares one or more type parameters in angle brackets before the parameter list:

```typescript
function identity<T>(value: T): T {
  return value;
}

const str = identity("hello");  // T inferred as string
const num = identity(42);       // T inferred as number
const explicit = identity<boolean>(true); // explicit type argument
```

TypeScript infers `T` from the argument type at the call site. Explicit type arguments are only needed when inference fails or when you want to restrict the inferred type.

### Multiple Type Parameters

```typescript
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}

const p = pair("key", 100); // [string, number]
```

### Generic Arrow Functions in TSX Files

In `.tsx` files, `<T>` is ambiguous with JSX syntax. Use a trailing comma or a constraint to disambiguate:

```typescript
// Fails in .tsx — looks like JSX
const identity = <T>(value: T): T => value;

// Works: trailing comma
const identity = <T,>(value: T): T => value;

// Works: constraint (even a trivial one)
const identity = <T extends unknown>(value: T): T => value;
```

## Generic Constraints

### extends Constraints

Use `extends` to restrict which types a type parameter can be:

```typescript
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}

getLength("hello");       // OK
getLength([1, 2, 3]);    // OK
getLength({ length: 5 }); // OK
getLength(42);            // Error: number has no 'length' property
```

### keyof Constraint

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };
const name = getProperty(user, "name"); // string
const age = getProperty(user, "age");   // number
getProperty(user, "email");             // Error: not a key of user
```

This is the safe, generic alternative to `obj[key as keyof typeof obj]`.

### Constraining to Another Type Parameter

```typescript
function merge<T, U extends Partial<T>>(target: T, source: U): T & U {
  return { ...target, ...source };
}
```

## Generic Interfaces

```typescript
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<void>;
}

interface User {
  id: string;
  name: string;
  email: string;
}

class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User | null> { /* ... */ }
  async findAll(): Promise<User[]> { /* ... */ }
  async save(user: User): Promise<User> { /* ... */ }
  async delete(id: string): Promise<void> { /* ... */ }
}
```

## Generic Type Aliases

```typescript
type Optional<T> = T | null | undefined;
type Nullable<T> = T | null;
type MaybePromise<T> = T | Promise<T>;

type AsyncResult<T, E extends Error = Error> =
  | { success: true; data: T }
  | { success: false; error: E };
```

## Generic Classes

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T {
    const item = this.items.pop();
    if (item === undefined) throw new Error("Stack is empty");
    return item;
  }

  peek(): T {
    if (this.items.length === 0) throw new Error("Stack is empty");
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
const top = numberStack.pop(); // number
```

## Default Type Parameters

Type parameters can have defaults, just like function parameters:

```typescript
interface ApiResponse<T = unknown, E = string> {
  data: T;
  error: E | null;
  status: number;
}

// Without explicit type argument, defaults apply
const response: ApiResponse = { data: null, error: null, status: 200 };

// With explicit types
const userResponse: ApiResponse<User, ApiError> = { /* ... */ };
```

Defaults are particularly useful in library types to avoid forcing users to specify types they don't care about.

## Generic Utility Patterns

### Wrapping and Unwrapping Types

```typescript
// Extract the value type from a Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;

type UserPromise = Promise<User>;
type ResolvedUser = Awaited<UserPromise>; // User

// Similarly for arrays
type ElementType<T extends readonly unknown[]> = T[number];

const items = ["a", "b", "c"] as const;
type Item = ElementType<typeof items>; // "a" | "b" | "c"
```

### Builder Pattern with Generics

```typescript
class QueryBuilder<T extends object> {
  private filters: Partial<T> = {};

  where<K extends keyof T>(key: K, value: T[K]): this {
    this.filters[key] = value;
    return this;
  }

  build(): Partial<T> {
    return this.filters;
  }
}

interface Product {
  id: number;
  name: string;
  category: string;
}

const query = new QueryBuilder<Product>()
  .where("category", "electronics")
  .where("name", "Laptop")
  .build();
```

### Generic Event Emitter

```typescript
type EventMap = Record<string, unknown>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners: Partial<{
    [K in keyof Events]: Array<(data: Events[K]) => void>;
  }> = {};

  on<K extends keyof Events>(
    event: K,
    listener: (data: Events[K]) => void
  ): void {
    (this.listeners[event] ??= []).push(listener);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners[event]?.forEach(fn => fn(data));
  }
}

interface AppEvents {
  userCreated: { userId: string; email: string };
  orderPlaced: { orderId: string; total: number };
}

const emitter = new TypedEventEmitter<AppEvents>();
emitter.on("userCreated", ({ userId, email }) => {
  // userId: string, email: string — fully typed
});
emitter.emit("userCreated", { userId: "1", email: "a@b.com" });
emitter.emit("userCreated", { userId: "1" }); // Error: missing 'email'
```

## Variance: Covariance and Contravariance

Generics in TypeScript are structurally checked, with variance inferred from usage:

```typescript
// Covariant: T appears only in output position
interface Producer<T> {
  produce(): T;
}

// A Producer<Dog> is assignable to Producer<Animal> if Dog extends Animal
// (you can use a more specific producer where a broader one is expected)

// Contravariant: T appears only in input position
interface Consumer<T> {
  consume(value: T): void;
}

// A Consumer<Animal> is assignable to Consumer<Dog>
// (you can use a more general consumer where a specific one is expected)
```

TypeScript 4.7 introduced explicit variance annotations:

```typescript
interface Provider<out T> {  // covariant
  get(): T;
}

interface Acceptor<in T> {   // contravariant
  set(value: T): void;
}
```

## Comparison Table

| Feature | Use Case | Example |
|---------|----------|---------|
| Generic function | Reusable logic with type preservation | `identity<T>`, `map<T, U>` |
| Generic constraint | Restrict acceptable types | `T extends { length: number }` |
| `keyof` constraint | Type-safe property access | `K extends keyof T` |
| Default type param | Optional type arguments | `<T = unknown>` |
| Generic class | Type-parameterized data structures | `Stack<T>`, `Map<K, V>` |
| Generic interface | Contracts across multiple implementations | `Repository<T>` |
| Variance annotation | Explicit subtype relationship control | `out T`, `in T` |

## Best Practices

### 1. Name Type Parameters Meaningfully

```typescript
// OK for simple cases
function identity<T>(value: T): T { ... }

// Prefer descriptive names in complex contexts
function transform<TInput, TOutput>(
  value: TInput,
  transform: (input: TInput) => TOutput
): TOutput { ... }
```

### 2. Don't Over-Generify

Only introduce a type parameter when you actually need to relate types. If the parameter is used only once, `unknown` or a concrete type is better:

```typescript
// Unnecessary generic — T is never used to relate types
function logValue<T>(value: T): void {
  console.log(value);
}

// Better
function logValue(value: unknown): void {
  console.log(value);
}
```

### 3. Use Constraints to Fail Fast

Apply the narrowest constraint that satisfies your implementation. This surfaces errors at the call site immediately:

```typescript
// Too permissive — error may occur deep inside
function sortItems<T>(items: T[]): T[] {
  return items.sort(); // May not work for all T
}

// Better — requires sortable items
function sortItems<T extends string | number>(items: T[]): T[] {
  return items.sort((a, b) => (a < b ? -1 : a > b ? 1 : 0));
}
```

### 4. Prefer Generic Functions over Overloads When Possible

```typescript
// Verbose: requires multiple overloads
function wrap(value: string): string[];
function wrap(value: number): number[];
function wrap(value: any): any[] {
  return [value];
}

// Clean: single generic function
function wrap<T>(value: T): T[] {
  return [value];
}
```

### 5. Use `infer` for Complex Generic Utilities

```typescript
// Extract the first parameter type of a function
type FirstParam<T extends (...args: any) => any> =
  T extends (first: infer P, ...rest: any[]) => any ? P : never;

type F = (x: string, y: number) => void;
type P = FirstParam<F>; // string
```

## Interview Questions

### Q1: What is the difference between `any` and a generic type parameter?

**Answer:** `any` opts out of type checking entirely — TypeScript stops tracking the type and allows any operation on it. A generic type parameter preserves type information across the function. When you write `function identity<T>(value: T): T`, TypeScript knows the return type is the same as the argument type. With `any`, `function identity(value: any): any` tells TypeScript nothing — the return type is `any`, losing all inference at the call site. Generics enable type safety while maintaining flexibility; `any` abandons type safety.

### Q2: When would you use a generic constraint vs an overload?

**Answer:** Use generic constraints when the implementation is uniform across the acceptable types and you want to express a relationship between types (e.g., input and output are the same type). Use overloads when the behavior or return type differs meaningfully between input types, or when TypeScript's inference cannot express the relationship generically. In general, prefer generics because they are less code, more composable, and do not require duplication of the implementation.

### Q3: What does `T extends keyof U` mean?

**Answer:** It means the type parameter `T` must be a key of `U` — that is, `T` is constrained to the union of `U`'s property names. This is the foundation of type-safe property access. For example, `function get<T, K extends keyof T>(obj: T, key: K): T[K]` guarantees that `key` is always a valid property of `obj`, and the return type is exactly `T[K]` — the type at that specific key. Without this constraint, accessing `obj[key]` would be unsafe.

### Q4: What is type parameter inference and when does it fail?

**Answer:** Type parameter inference is TypeScript's ability to automatically determine the type argument from the call-site context — typically from argument types. Inference works when TypeScript can unambiguously derive the type from the arguments provided. It can fail when: the type parameter appears only in the return type and not in any parameter (inference requires input), when there are conflicting inference candidates from multiple arguments, or when generic functions are passed as higher-order arguments and the context does not provide enough information. When inference fails, you must provide explicit type arguments.

### Q5: What are variance annotations (`in`/`out`) in TypeScript 4.7 and why do they matter?

**Answer:** Variance annotations let you explicitly declare whether a type parameter is covariant (`out T` — T appears in output position), contravariant (`in T` — T appears in input position), or invariant. Without annotations, TypeScript infers variance by analyzing all usages, which can be expensive in large codebases and sometimes inaccurate for complex types. Explicit annotations serve as both documentation and a compiler hint that speeds up type checking. Covariant type parameters support subtype substitution in one direction; contravariant support it in the other. Understanding variance is important when designing generic library APIs.

## Common Pitfalls

### 1. Returning a More Specific Type than the Type Parameter

```typescript
// Bug: the return type is not actually T
function ensureArray<T>(value: T | T[]): T[] {
  if (Array.isArray(value)) {
    return value;
  }
  return [value]; // Fine — T[] is correct
}

// But this breaks:
function firstOrDefault<T>(arr: T[], fallback: T): T {
  return arr[0] ?? fallback; // OK — returns T
}

// Subtle bug: the cast hides that arr[0] could be undefined
function first<T>(arr: T[]): T {
  return arr[0]; // Unsafe: T might be inferred as string, but arr could be empty
  // Better: return arr[0] as T; — explicit, honest about the assertion
  // Best: return T | undefined and let callers handle it
}
```

### 2. Losing Type Information with `extends any[]`

```typescript
function mapArray<T extends any[]>(arr: T): T {
  return arr.map(x => x) as T; // TypeScript may lose tuple info
}

// If you want to preserve tuple types:
function mapTuple<T extends readonly unknown[]>(
  arr: [...T]
): { [K in keyof T]: T[K] } {
  return arr.map(x => x) as any;
}
```

### 3. Generic Classes and Static Members

Type parameters are not available on static members:

```typescript
class Container<T> {
  static defaultValue: T; // Error: static member cannot reference class type parameter

  // Fix: use a concrete type or a separate generic method
  static create<U>(value: U): Container<U> {
    return new Container<U>();
  }
}
```

### 4. Incorrect Constraint Can Make a Function Uncallable

```typescript
// Over-constrained: impossible to satisfy in practice
function merge<T extends string & number>(a: T, b: T): T { ... }
// T extends `never` — this function can never be called
```

### 5. Type Parameters Captured by Closures

```typescript
function makeAdder<T extends number>(x: T) {
  return (y: T) => x + y; // Fine
}

const add5 = makeAdder(5);
add5(3); // Works: both T inferred as number
```

## Resources

### Official Documentation
- [TypeScript Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TypeScript Handbook: Indexed Access Types](https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html)
- [TypeScript 4.7 Release Notes: Variance Annotations](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html)

### Articles and Guides
- [Total TypeScript: Generics Workshop](https://www.totaltypescript.com/workshops/typescript-generics)
- [TypeScript Deep Dive: Generics](https://basarat.gitbook.io/typescript/type-system/generics)
- [Effective TypeScript: Items 26-29 on Generics](https://effectivetypescript.com/)

### Tools
- [TypeScript Playground](https://www.typescriptlang.org/play): Test generic inference live
- [typescript-exercises](https://typescript-exercises.github.io/): Hands-on generic challenges
