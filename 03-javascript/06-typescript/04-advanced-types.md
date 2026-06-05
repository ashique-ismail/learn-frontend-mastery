# Advanced Types: Conditional, Mapped, and Template Literal Types

## The Idea

**In plain English:** Advanced types in TypeScript let you write rules that automatically create new types by transforming or filtering existing ones — like a smart template that can inspect a type and produce a customized version of it based on conditions you define.

**Real-world analogy:** Think of a school that auto-generates a personalised report card format for each student based on their grade level — younger students get simpler categories, older ones get detailed subject breakdowns, and all names are formatted with a "Student: " prefix automatically.

- The grade-level check = a conditional type (branch on what kind of type you have)
- The auto-prefixed "Student: " label = a template literal type (building new string types from parts)
- The rule that rewrites every category name = a mapped type (transforming all keys of an object type)

---

## Overview

TypeScript's type system is Turing-complete at the type level. Conditional types, mapped types, and template literal types are the primary mechanisms that make this possible. These features let you compute types from other types — transforming shapes, filtering union members, generating string unions, and encoding complex logic entirely within the type system. They underpin every utility type in the standard library and are essential for writing sophisticated library APIs.

## Conditional Types

### Basic Syntax

A conditional type has the form `T extends U ? TrueType : FalseType`. At evaluation time, TypeScript checks whether `T` is assignable to `U` and resolves to one of the two branches:

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;   // true
type B = IsString<number>;   // false
type C = IsString<"hello">;  // true (string literal extends string)
```

### Distributive Conditional Types

When the checked type is a naked type parameter (not wrapped in anything), conditional types distribute over union members:

```typescript
type IsArray<T> = T extends any[] ? "array" : "not array";

type D = IsArray<string | number[]>;
// = IsArray<string> | IsArray<number[]>
// = "not array" | "array"

// Without distribution (wrapped in a tuple):
type IsArrayStrict<T> = [T] extends [any[]] ? "array" : "not array";
type E = IsArrayStrict<string | number[]>; // "not array" (string | number[] does not extend any[])
```

This distinction is critical. Wrapping in `[T]` prevents distribution and checks the union as a whole.

### `infer` Keyword

`infer` introduces a new type variable that TypeScript infers from the match. It only works inside `extends` clauses of conditional types:

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type UnpackPromise<T> = T extends Promise<infer U> ? U : T;

type ArrayElement<T> = T extends (infer E)[] ? E : never;

// Chaining infer patterns
type HeadAndTail<T extends any[]> = T extends [infer Head, ...infer Tail]
  ? { head: Head; tail: Tail }
  : never;

type HT = HeadAndTail<[string, number, boolean]>;
// { head: string; tail: [number, boolean] }
```

### Multiple `infer` Positions

```typescript
// Infer both the parameter and return type of a function
type FunctionParts<T extends (...args: any[]) => any> = T extends (
  ...args: infer P
) => infer R
  ? { params: P; returns: R }
  : never;

type Parts = FunctionParts<(x: string, y: number) => boolean>;
// { params: [x: string, y: number]; returns: boolean }
```

### Recursive Conditional Types

TypeScript 4.1+ supports conditional types that reference themselves:

```typescript
type Flatten<T> = T extends Array<infer Item> ? Flatten<Item> : T;

type Nested = number[][][];
type Flat = Flatten<Nested>; // number

// Deep readonly
type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

### Conditional Types on Union Members

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
// Distributes: removes null and undefined from each union member

type FilterStrings<T> = T extends string ? T : never;
type S = FilterStrings<string | number | boolean>; // string
```

## Mapped Types

### Basic Mapped Type

A mapped type iterates over a union of keys and produces a new object type:

```typescript
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

type Partial<T> = {
  [K in keyof T]?: T[K];
};

type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};
```

### Modifier Control: `+` and `-`

The `+` and `-` prefixes on modifiers let you add or remove `readonly` and `?`:

```typescript
// Remove readonly
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Remove optional
type Required<T> = {
  [K in keyof T]-?: T[K];
};

// Both together
type MutableRequired<T> = {
  -readonly [K in keyof T]-?: T[K];
};
```

### Remapping Keys with `as`

TypeScript 4.1 introduced key remapping, allowing you to transform keys via a template literal or conditional type:

```typescript
// Prefix all keys with "get"
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  id: number;
  name: string;
}

type UserGetters = Getters<User>;
// { getId: () => number; getName: () => string }
```

### Filtering Keys with `as` + `never`

When the remapped key resolves to `never`, the property is excluded:

```typescript
// Keep only string-valued properties
type StringProperties<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

interface Config {
  host: string;
  port: number;
  name: string;
  active: boolean;
}

type StringConfig = StringProperties<Config>;
// { host: string; name: string }
```

### Mapped Types Over Unions

```typescript
type UnionToObject<T extends string> = {
  [K in T]: K;
};

type RouteMap = UnionToObject<"home" | "about" | "contact">;
// { home: "home"; about: "about"; contact: "contact" }
```

### Event Handlers Pattern

```typescript
type EventHandlers<T extends Record<string, unknown>> = {
  [K in keyof T as `on${Capitalize<string & K>}`]?: (event: T[K]) => void;
};

interface DOMEvents {
  click: MouseEvent;
  focus: FocusEvent;
  input: InputEvent;
}

type Handlers = EventHandlers<DOMEvents>;
// { onClick?: (event: MouseEvent) => void; onFocus?: ...; onInput?: ... }
```

## Template Literal Types

### Basic Syntax

Template literal types mirror JavaScript template literals at the type level:

```typescript
type World = "world";
type Greeting = `hello ${World}`;   // "hello world"

type Prop = "color" | "size";
type CSSProp = `--${Prop}`;        // "--color" | "--size"
```

When used with unions, they distribute and produce the cross-product:

```typescript
type Side = "top" | "right" | "bottom" | "left";
type Property = "margin" | "padding";

type BoxModel = `${Property}-${Side}`;
// "margin-top" | "margin-right" | ... | "padding-left" (8 members)
```

### Inferring from Template Literals

`infer` works inside template literal types:

```typescript
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
    ? Param
    : never;

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

### Type-Safe Event System

```typescript
type ListenerMap<T extends Record<string, unknown>> = {
  [K in keyof T]: (payload: T[K]) => void;
};

type On<T extends Record<string, unknown>> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: (
    callback: (payload: T[K]) => void
  ) => void;
};
```

### Getter/Setter Pairing

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type Accessors<T> = Getters<T> & Setters<T>;

interface Config {
  host: string;
  port: number;
}

type ConfigAccessors = Accessors<Config>;
// { getHost: () => string; setHost: (value: string) => void; getPort: () => number; setPort: (value: number) => void }
```

## Combining All Three

These features compose naturally for powerful type-level computation:

```typescript
// Type-safe path accessor
type PathValue<T, Path extends string> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? PathValue<T[Key], Rest>
      : never
    : Path extends keyof T
    ? T[Path]
    : never;

interface DeepConfig {
  server: {
    host: string;
    port: number;
  };
  database: {
    url: string;
    pool: {
      size: number;
    };
  };
}

type HostType = PathValue<DeepConfig, "server.host">;      // string
type PoolSize = PathValue<DeepConfig, "database.pool.size">; // number
```

```typescript
// Flatten nested keys to dot notation paths
type DotPaths<T, Prefix extends string = ""> = {
  [K in keyof T]: T[K] extends object
    ? DotPaths<T[K], `${Prefix}${string & K}.`>
    : `${Prefix}${string & K}`;
}[keyof T];

type Paths = DotPaths<DeepConfig>;
// "server.host" | "server.port" | "database.url" | "database.pool.size"
```

## Comparison Table

| Feature | Purpose | Key Syntax |
|---------|---------|------------|
| Conditional type | Branch on type assignability | `T extends U ? A : B` |
| `infer` | Extract type from a pattern | `infer R` inside `extends` clause |
| Distributive conditional | Apply to each union member | Naked type param in `extends` |
| Mapped type | Transform object type keys/values | `[K in keyof T]: ...` |
| Key remapping | Rename or filter keys | `[K in keyof T as ...]` |
| `+`/`-` modifiers | Add/remove `readonly`/`?` | `-readonly`, `-?` |
| Template literal type | Compute string union types | `` `prefix${T}` `` |
| Recursive conditional | Process nested/recursive types | Self-referential conditional |

## Best Practices

### 1. Wrap Type Params to Prevent Unwanted Distribution

```typescript
// If you want to check if the entire union extends something:
type IsNever<T> = [T] extends [never] ? true : false;
// Without wrapping: T extends never always distributes to `never`

type Test1 = IsNever<never>;  // true
type Test2 = IsNever<string>; // false
```

### 2. Use Conditional Types to Create Useful Extraction Utilities

```typescript
type EventsOf<T> = {
  [K in keyof T]: T[K] extends (...args: any[]) => void ? K : never;
}[keyof T];

// Extracts only the method names of type T
```

### 3. Limit Recursion Depth

TypeScript limits recursive type instantiation depth (typically around 100). For deep structures, consider a different approach:

```typescript
// If recursive types hit limits, add a depth counter:
type DeepPartial<T, Depth extends number[] = []> =
  Depth["length"] extends 5
    ? T // Stop recursion at depth 5
    : T extends object
    ? { [K in keyof T]?: DeepPartial<T[K], [0, ...Depth]> }
    : T;
```

### 4. Test Your Types with `type-challenges` Patterns

Use `type-level` assertions to verify type behavior:

```typescript
type Expect<T extends true> = T;
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;

type TestFlatten = Expect<Equal<Flatten<string[][][]>, string>>; // Passes if true
```

### 5. Keep Complex Types Readable via Intermediate Aliases

```typescript
// Unreadable
type X<T> = { [K in keyof T as T[K] extends string ? `get${Capitalize<string & K>}` : never]: () => T[K] };

// Readable
type StringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];
type GetterName<K extends string> = `get${Capitalize<K>}`;
type StringGetters<T> = { [K in StringKeys<T> & string as GetterName<K>]: () => T[K] };
```

---

## Supplementary: The `satisfies` Operator (TypeScript 4.9+)

The `satisfies` operator validates that an expression matches a type **without widening** the inferred type. It's the fix for the classic tension between type safety and inference.

### The Problem

```typescript
// Option A: type annotation — safe but loses literal inference
const palette: Record<string, string[]> = {
  red:   ['#f00', 'rgb(255,0,0)'],
  green: ['#0f0'],
};
palette.red;   // string[]  — correct
palette.mauve; // string[]  — no error! key isn't checked

// Option B: no annotation — keeps literals but unsafe
const palette = {
  red:   ['#f00', 'rgb(255,0,0)'],
  green: ['#0f0'],
};
palette.red;   // string[]  — correct
palette.mauve; // error     — good!
// But nothing checks that values are string[]
```

### The Solution: `satisfies`

```typescript
type Colors = 'red' | 'green' | 'blue';

const palette = {
  red:   ['#f00', 'rgb(255,0,0)'],
  green: ['#0f0'],
  blue:  '#00f',
} satisfies Record<Colors, string | string[]>;

// ✓ Type is checked — 'mauve' would be a compile error
// ✓ Literal types are preserved — palette.red is string[], not string | string[]
// ✓ palette.blue is 'string', not 'string | string[]'

palette.red.at(0);    // string — .at() works, type is string[]
palette.blue.toUpperCase(); // string — not string[]

// Would error: 'mauve' is not in Colors
const bad = { mauve: '#f0f' } satisfies Record<Colors, string>; // ❌
```

### `satisfies` vs Type Annotation vs `as const`

```typescript
type Config = { host: string; port: number };

// Annotation: widens, loses literals
const a: Config = { host: 'localhost', port: 3000 };
// a.host is 'string', not 'localhost'

// satisfies: checks shape, keeps literals
const b = { host: 'localhost', port: 3000 } satisfies Config;
// b.host is 'localhost' (literal), b.port is 3000 (literal)

// as const: fully literal, no shape validation
const c = { host: 'localhost', port: 3000 } as const;
// c.host is 'localhost' — but no check that it matches Config

// satisfies + as const: shape-checked AND fully literal
const d = { host: 'localhost', port: 3000 } as const satisfies Config;
```

### Practical Use Cases

```typescript
// Route config: each route checked, path literals preserved
type Route = { path: string; component: React.ComponentType };

const routes = [
  { path: '/home',    component: HomePage },
  { path: '/profile', component: ProfilePage },
] satisfies Route[];

// routes[0].path is '/home' (literal), not string

// Event map: exhaustive event types preserved
type Events = {
  click: MouseEvent;
  keydown: KeyboardEvent;
};

const handlers = {
  click:   (e) => console.log(e.clientX),  // e is MouseEvent — inferred!
  keydown: (e) => console.log(e.key),      // e is KeyboardEvent — inferred!
} satisfies { [K in keyof Events]: (e: Events[K]) => void };

// Enum-like objects: values are literal, keys are checked
const Direction = {
  Up:    'UP',
  Down:  'DOWN',
  Left:  'LEFT',
  Right: 'RIGHT',
} as const satisfies Record<string, string>;

// Direction.Up is 'UP' (literal), not string
```

---

## Interview Questions

### Q1: What is the difference between a distributive and non-distributive conditional type?

**Answer:** A conditional type is distributive when the checked type is a naked type parameter — not wrapped in a tuple, array, or object. When TypeScript evaluates `T extends U ? A : B` and `T` is a bare type parameter, it distributes across union members, evaluating the condition separately for each member and producing a union of results. Wrapping prevents distribution: `[T] extends [U] ? A : B` checks the union as a whole. This distinction matters for checks like `IsNever<T>` — without wrapping, `never extends X` distributes over zero members and always produces `never`, which is rarely desired.

### Q2: How does `infer` work and where can it be used?

**Answer:** `infer` introduces a new type variable that TypeScript infers from a pattern match. It can only appear in the `extends` clause of a conditional type, not in the true/false branches or elsewhere. When the conditional type matches, `infer R` captures the matched type and makes `R` available in the true branch. Multiple `infer` positions can appear in the same `extends` clause. Common uses: extracting return types (`infer R` from function types), unwrapping Promise values, extracting tuple elements, and parsing template literal patterns.

### Q3: What does remapping keys with `as` let you do that wasn't possible before?

**Answer:** Key remapping (TypeScript 4.1+) lets you transform or filter property names within a mapped type. Before `as`, you could only use `keyof T` as-is — you couldn't rename keys. With `as`, you can: rename keys using template literals (e.g., `get${Capitalize<K>}`), exclude keys by mapping them to `never`, create new keys derived from values or positions, and combine mapped types with string manipulation types. This enables patterns like auto-generating getter/setter pairs, event handler names, and type-safe proxy interfaces from existing types.

### Q4: When would you choose a conditional type over a function overload?

**Answer:** Use a conditional type when the return type of a function should depend on the input type in a way that TypeScript can compute statically, and when you want a single implementation to cover multiple cases. Conditional types express relationships between input and output types clearly and compose with generics. Use overloads when the behavior genuinely differs between cases (not just the type), when the types are unrelated enough that a conditional type would be awkward, or when the number of cases is small and explicit. Conditional types are generally preferred because they avoid repetition and work better with inference.

### Q5: What is the cross-product behavior of template literal types with unions?

**Answer:** When a template literal type includes union types in interpolation positions, TypeScript distributes across all combinations, producing the cross-product as a new union. For example, `` `${'a' | 'b'}${'1' | '2'}` `` produces `"a1" | "a2" | "b1" | "b2"`. This is extremely useful for generating typed string unions from smaller building blocks — routing parameters, CSS property names, event handler names, etc. — without writing each combination manually. With large unions in multiple positions, the resulting union can grow exponentially, which can slow down compilation.

## Common Pitfalls

### 1. `never` Disappears in Unions

```typescript
type ExcludeNumber<T> = T extends number ? never : T;

type Result = ExcludeNumber<string | number | boolean>;
// string | boolean (never is removed from union automatically)

// But if T is exactly never:
type IsNever_Wrong<T> = T extends never ? true : false;
type Test = IsNever_Wrong<never>; // never (not true!)
// Fix: [T] extends [never] ? true : false
```

### 2. Circular Mapped Types Cause Infinite Recursion

```typescript
// This causes a "Type alias 'Bad' circularly references itself" error
type Bad<T> = {
  [K in keyof T]: T[K] extends object ? Bad<T[K]> : T[K]; // Unconstrained recursion
};
```

### 3. Template Literal Types Are Case-Sensitive

```typescript
type Greeting = `Hello ${string}`;

const g1: Greeting = "Hello World";   // OK
const g2: Greeting = "hello World";   // Error: lowercase 'h'
const g3: Greeting = "Hello ";        // OK (empty string extends string)
```

### 4. Key Remapping Does Not Preserve Optional/Readonly Modifiers by Default

```typescript
type Optional<T> = { [K in keyof T]?: T[K] };

// If you remap keys, modifiers from the original may be lost
type Renamed<T> = { [K in keyof T as `_${string & K}`]: T[K] };

// The ? modifier is preserved only when using `as`, but only if the source had it
// and you don't explicitly change it
```

### 5. Conditional Types Are Not Always Evaluated Eagerly

When a conditional type's checked type is still a generic (deferred), TypeScript may not simplify it until the type is instantiated. This can lead to unexpected assignability errors in intermediate types:

```typescript
function test<T>(value: T): T extends string ? number : boolean {
  if (typeof value === "string") {
    return 42; // Error even though runtime is correct — type is deferred
  }
  return true; // Same error
}

// Fix: use a cast
function test<T>(value: T): T extends string ? number : boolean {
  if (typeof value === "string") {
    return 42 as any;
  }
  return true as any;
}
```

## Resources

### Official Documentation
- [TypeScript Handbook: Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [TypeScript Handbook: Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript Handbook: Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [TypeScript 4.1 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html)

### Articles and Guides
- [Total TypeScript: Advanced Types Workshop](https://www.totaltypescript.com/workshops/advanced-typescript-patterns)
- [type-challenges](https://github.com/type-challenges/type-challenges): Progressive type system exercises
- [Effective TypeScript: Items 50-52 on Conditional Types](https://effectivetypescript.com/)

### Tools
- [TypeScript Playground](https://www.typescriptlang.org/play)
- [ts-toolbelt](https://github.com/millsp/ts-toolbelt): Advanced type utilities built on these foundations
