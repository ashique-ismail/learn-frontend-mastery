# Basic Types, Unions, Intersections, and Literals in TypeScript

## Overview

TypeScript's type system is structural and gradually typed, built on top of JavaScript's runtime primitives. Understanding how primitive types, union types, intersection types, and literal types interact is foundational to writing safe, expressive TypeScript code. These constructs form the vocabulary of the type system and appear in every codebase — from simple utilities to complex library APIs.

## Primitive Types

TypeScript mirrors JavaScript's primitive types and adds `unknown`, `never`, and `void` on top.

### Core Primitives

```typescript
const name: string = "Alice";
const age: number = 30;
const isAdmin: boolean = true;
const id: bigint = 9007199254740993n;
const sym: symbol = Symbol("id");
const nothing: null = null;
const notYet: undefined = undefined;
```

TypeScript infers types when you initialize variables at declaration, so explicit annotations are usually redundant here. The above is intentionally verbose for illustration.

### void

`void` represents the absence of a return value. It is mainly used for function return types:

```typescript
function logMessage(msg: string): void {
  console.log(msg);
  // returning undefined is allowed; returning a value is not
}
```

A variable typed `void` can only hold `undefined`. This is distinct from functions that return `undefined` explicitly — though in practice the distinction rarely matters outside of callbacks.

## Arrays and Tuples

### Arrays

```typescript
const scores: number[] = [90, 85, 92];
const tags: Array<string> = ["typescript", "web"];

// Readonly arrays prevent mutation
const config: readonly string[] = ["a", "b"];
config.push("c"); // Error: Property 'push' does not exist on type 'readonly string[]'
```

### Tuples

Tuples encode fixed-length arrays with known types at specific positions:

```typescript
type Point = [number, number];
const origin: Point = [0, 0];

// Named tuple elements (TypeScript 4.0+)
type Range = [start: number, end: number];

// Optional elements at the tail
type OptionalLabel = [value: string, label?: string];

// Rest elements
type StringsAndNumber = [string, ...string[], number];
```

Tuples provide the type-level equivalent of structured destructuring:

```typescript
function parseCoord(input: string): [number, number] {
  const [x, y] = input.split(",").map(Number);
  return [x, y];
}

const [lat, lng] = parseCoord("51.5,-0.1");
```

## Object Types

### Inline Object Types

```typescript
function greet(user: { name: string; age: number }): string {
  return `Hello, ${user.name}`;
}
```

### Interfaces vs Type Aliases

Both describe object shapes, but they differ in extensibility and capability:

```typescript
// Interface — can be extended and merged (declaration merging)
interface User {
  id: number;
  name: string;
}

interface Admin extends User {
  role: "admin";
}

// Type alias — more flexible, supports unions/intersections inline
type Coordinate = { x: number; y: number };
type Named = { name: string };
type NamedCoordinate = Coordinate & Named;
```

Declaration merging is exclusive to interfaces:

```typescript
interface Window {
  myPlugin: () => void;
}

// Now window.myPlugin is recognized by TypeScript
```

### Optional and Readonly Properties

```typescript
interface Config {
  host: string;
  port?: number;           // optional
  readonly apiKey: string; // cannot be reassigned after creation
}
```

### Index Signatures

```typescript
interface StringMap {
  [key: string]: string;
}

// More precise: only string values, but known keys can also be typed
interface EventMap {
  [event: string]: (...args: unknown[]) => void;
  click: (e: MouseEvent) => void; // specific key must be compatible
}
```

## Union Types

A union type means a value can be one of several types. It is one of TypeScript's most expressive features.

```typescript
type StringOrNumber = string | number;

function formatId(id: string | number): string {
  if (typeof id === "number") {
    return id.toFixed(0);
  }
  return id.trim();
}
```

### Discriminated Unions

The most important pattern in TypeScript — a tagged union where a shared literal property acts as a discriminant:

```typescript
type Circle = {
  kind: "circle";
  radius: number;
};

type Rectangle = {
  kind: "rectangle";
  width: number;
  height: number;
};

type Triangle = {
  kind: "triangle";
  base: number;
  height: number;
};

type Shape = Circle | Rectangle | Triangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      // Exhaustiveness check — TypeScript narrows to `never` here
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${_exhaustive}`);
  }
}
```

The exhaustiveness check is a compile-time guarantee: if you add a new `Shape` variant and forget to handle it in `area`, TypeScript will error at the `never` assignment.

### Union Widening Pitfalls

```typescript
function getValue(): string | null {
  return Math.random() > 0.5 ? "value" : null;
}

const val = getValue();
// val.toUpperCase(); // Error: Object is possibly 'null'

// Must narrow first:
if (val !== null) {
  console.log(val.toUpperCase()); // safe
}
```

## Intersection Types

An intersection type combines multiple types into one, requiring all properties from all members:

```typescript
type Serializable = {
  serialize(): string;
};

type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type AuditedEntity = Serializable & Timestamped;

// Must implement all members
const record: AuditedEntity = {
  createdAt: new Date(),
  updatedAt: new Date(),
  serialize() {
    return JSON.stringify(this);
  }
};
```

### Intersection with Primitives

Intersecting primitive types that are incompatible results in `never`:

```typescript
type Impossible = string & number; // never
```

This is used intentionally in some advanced patterns (branded/opaque types).

### Intersection vs Interface Extension

They are semantically similar but behave differently for conflicting properties:

```typescript
interface A {
  foo: string;
}

interface B extends A {
  foo: number; // Error: Types of property 'foo' are incompatible
}

// With intersection, conflicting properties narrow to `never`
type C = { foo: string } & { foo: number };
// C.foo is `never` — the type exists but no value can satisfy it
```

## Literal Types

A literal type is a type whose only value is exactly the specified literal:

```typescript
type Direction = "north" | "south" | "east" | "west";
type StatusCode = 200 | 201 | 400 | 404 | 500;
type Truthy = true;
```

### String Literal Types

```typescript
function setAlignment(alignment: "left" | "center" | "right"): void {
  // ...
}

setAlignment("left");   // OK
setAlignment("middle"); // Error: Argument of type '"middle"' is not assignable...
```

### Numeric Literal Types

```typescript
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

function rollDice(): DiceRoll {
  return (Math.floor(Math.random() * 6) + 1) as DiceRoll;
}
```

### Const Assertions

Without a `const` assertion, TypeScript widens literal types:

```typescript
const direction = "north"; // Type: "north" (literal, inferred from const)
let mutableDir = "north";  // Type: string (widened for let)

// Explicitly narrow a mutable binding:
const config = {
  method: "GET",  // Type: string — widened
} as const;

// Now all properties are readonly literal types:
// config.method: "GET"
```

`as const` recursively makes all properties `readonly` and narrows all literals:

```typescript
const routes = ["home", "about", "contact"] as const;
// Type: readonly ["home", "about", "contact"]

type Route = (typeof routes)[number]; // "home" | "about" | "contact"
```

This pattern is commonly used to derive union types from arrays without duplication.

## The `object`, `Object`, and `{}` Types

These three are confusingly different:

```typescript
// `object` — any non-primitive type
let obj: object = { a: 1 };
obj = "hello"; // Error

// `Object` — the Object prototype interface; accepts almost everything
let o: Object = 42; // OK (primitives have Object methods via boxing)

// `{}` — any non-nullish value (widest non-nullish type)
let anything: {} = 42;     // OK
let anything2: {} = null;  // Error
```

In practice, prefer `object` over `Object`, and prefer specific interfaces over `{}`.

## Comparison Table

| Concept | When to Use | Key Behavior |
|---------|-------------|--------------|
| Union (`\|`) | Value is one of several types | Requires narrowing before accessing type-specific members |
| Intersection (`&`) | Value must satisfy all types | Combines properties; conflicts narrow to `never` |
| Literal type | Exact value constraint | Enables exhaustiveness checks and precise APIs |
| `as const` | Derive literal types from objects/arrays | Makes all values `readonly` and literal |
| Interface | Object shape with extension/merging | Supports declaration merging |
| Type alias | Unions, intersections, mapped types | More flexible, no declaration merging |
| Tuple | Fixed-length, positionally typed array | Enables named destructuring |

## Best Practices

### 1. Prefer Discriminated Unions over Optional Properties

```typescript
// Fragile — ambiguous what combination of optionals is valid
interface Result {
  data?: User;
  error?: string;
  loading?: boolean;
}

// Clear — mutually exclusive states
type Result =
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: string };
```

### 2. Use `as const` to Derive Types from Values

```typescript
const PERMISSIONS = ["read", "write", "admin"] as const;
type Permission = (typeof PERMISSIONS)[number];
// "read" | "write" | "admin"

// Adding a new permission automatically updates the type
```

### 3. Avoid Overusing Intersections for Object Composition

Intersections of large object types can slow down the TypeScript compiler. Prefer interface extension for simple inheritance:

```typescript
// Prefer this for simple composition
interface AdminUser extends User {
  role: "admin";
  permissions: string[];
}

// Reserve intersections for combining separately defined types
type AuditedAdmin = AdminUser & Auditable;
```

### 4. Prefer Named Tuple Elements for Public APIs

```typescript
// Hard to read — what is index 0?
function getRange(): [number, number] { ... }

// Self-documenting
function getRange(): [min: number, max: number] { ... }
```

### 5. Avoid `any` by Defaulting to `unknown` for External Data

```typescript
// Bad
function parseResponse(data: any): User {
  return data.user; // No safety
}

// Good
function parseResponse(data: unknown): User {
  if (!isUser(data)) throw new Error("Invalid response");
  return data;
}
```

## Interview Questions

### Q1: What is the difference between a type alias and an interface?

**Answer:** Both can describe object shapes, but they differ in three key ways. First, interfaces support declaration merging — you can define the same interface in multiple places and TypeScript merges them. Type aliases do not merge. Second, type aliases can express union types, intersection types, mapped types, and conditional types directly, while interfaces cannot. Third, interface extension (`extends`) gives more readable error messages and is preferred for object hierarchies. In practice: use interfaces for object shapes that may be extended, and type aliases for union types, function types, and complex type computations.

### Q2: What is a discriminated union and why is it useful?

**Answer:** A discriminated union (also called a tagged union or algebraic data type) is a union of types that share a common literal property — the discriminant. TypeScript uses the discriminant to narrow the type within control flow. This is useful because it models mutually exclusive states explicitly (e.g., loading/success/error), eliminates ambiguous optional fields, and allows exhaustiveness checking with the `never` type — if you miss a case in a switch, TypeScript errors at compile time.

### Q3: What does `as const` do, and how is it different from `readonly`?

**Answer:** `as const` is a const assertion that tells TypeScript to infer the narrowest possible type for an expression. It makes all properties `readonly` and narrows string/number/boolean values to their literal types. `readonly` is a modifier you apply to individual properties or array types explicitly. The key difference: `as const` is applied to expressions and affects the entire nested structure recursively; `readonly` is applied at the type level on specific properties. `as const` is commonly used to derive union types from arrays and to lock down configuration objects without writing explicit types.

### Q4: When does an intersection type produce `never`?

**Answer:** When the intersected types have conflicting primitive constraints. For example, `string & number` is `never` because no value can simultaneously be both a string and a number. More subtly, if two object types have a property with the same name but incompatible types — e.g., `{ foo: string } & { foo: number }` — the resulting type has `foo: never`. This is a compile-time type but it means no value can satisfy the property. The `never` type in intersections is sometimes used intentionally for branded/opaque types.

### Q5: What is the difference between `string[]` and `Array<string>`?

**Answer:** They are semantically identical. `string[]` is syntactic sugar for `Array<string>`. The choice is stylistic; many style guides prefer `string[]` for primitive element types and `Array<ComplexType>` for readability when the element type is long or complex. The important variant is `ReadonlyArray<string>` or `readonly string[]`, which prevents mutation methods like `push` and `pop` — useful for enforcing immutability at the type level.

## Common Pitfalls

### 1. Widening Literal Types Unexpectedly

```typescript
const status = { code: 200 }; // code: number (widened, NOT 200)
// To preserve the literal:
const status = { code: 200 } as const; // code: 200
// Or annotate explicitly:
const status: { code: 200 } = { code: 200 };
```

### 2. Confusing Union and Intersection Semantics

```typescript
// A | B means: a value that satisfies A OR B
// A & B means: a value that satisfies A AND B (has all properties of both)

// Common mistake: thinking & produces a "smaller" type
// Actually it requires MORE properties, not fewer
type Wide = { a: string } & { b: number };
const x: Wide = { a: "hello", b: 42 }; // Must have both
```

### 3. Forgetting That Optional Properties and Union with undefined Are Not the Same

```typescript
interface A {
  foo?: string; // foo is optional: can be absent OR undefined
}

interface B {
  foo: string | undefined; // foo is required but can be undefined
}

const a: A = {};             // OK
const b: B = {};             // Error: Property 'foo' is missing
const b2: B = { foo: undefined }; // OK
```

With `exactOptionalPropertyTypes` enabled in `tsconfig.json`, TypeScript distinguishes between `undefined` and absent even more strictly.

### 4. Index Signatures Override Specific Property Types

```typescript
interface Broken {
  name: string;
  [key: string]: number; // Error: 'name' must be compatible with index signature
}

// Fix: use a union in the index signature
interface Fixed {
  name: string;
  [key: string]: string | number;
}
```

### 5. Tuple vs Array Inference

```typescript
function swap(pair: [string, number]): [number, string] {
  return [pair[1], pair[0]];
}

// Without explicit annotation, TypeScript may infer (string | number)[]
const pair = ["hello", 42]; // Type: (string | number)[]
// Fix:
const pair: [string, number] = ["hello", 42];
// Or:
const pair = ["hello", 42] as [string, number];
```

## Resources

### Official Documentation
- [TypeScript Handbook: Basic Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript Handbook: Union and Intersection Types](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)
- [TypeScript Handbook: Literal Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-types)
- [TypeScript Handbook: Object Types](https://www.typescriptlang.org/docs/handbook/2/objects.html)

### Articles and Guides
- [TypeScript Deep Dive: Type System](https://basarat.gitbook.io/typescript/type-system)
- [Effective TypeScript: 62 Specific Ways to Improve Your TypeScript](https://effectivetypescript.com/)
- [Matt Pocock: Total TypeScript](https://www.totaltypescript.com/)

### Tools
- [TypeScript Playground](https://www.typescriptlang.org/play): Experiment with types interactively
- [ts-reset](https://github.com/total-typescript/ts-reset): Library that fixes common TypeScript type weaknesses
