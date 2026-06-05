# Declaration Files and Module Augmentation in TypeScript

## The Idea

**In plain English:** A declaration file is a special cheat sheet that tells TypeScript what a piece of JavaScript code looks like — what functions it has, what they accept, and what they return — without containing any actual running code. TypeScript reads these cheat sheets so it can warn you if you use a library the wrong way.

**Real-world analogy:** Imagine a restaurant menu at the front door. The menu describes every dish, its ingredients, and its price — but the menu itself is not food and you cannot eat it. The kitchen (the actual JavaScript library) does all the real cooking.

- The menu = the declaration file (`.d.ts`) describing what is available
- Each dish listing = a function or type signature telling you what inputs it takes and what you get back
- The kitchen = the JavaScript library that runs at runtime

---

## Overview

Declaration files (`.d.ts`) provide TypeScript type information for JavaScript code that has no types of its own — third-party libraries, auto-generated code, legacy JavaScript modules, and environment globals. Module augmentation extends existing declaration files to add or modify types without touching the original source. Together, these mechanisms form the bridge between TypeScript's type system and the JavaScript ecosystem.

## What Declaration Files Are

A `.d.ts` file contains only type declarations — no values, no runtime code. When TypeScript compiles a project, these files are consulted during type checking but produce no output:

```typescript
// user.d.ts
export interface User {
  id: number;
  name: string;
  email: string;
}

export function fetchUser(id: number): Promise<User>;
export function createUser(data: Omit<User, "id">): Promise<User>;
```

Declaration files appear in three contexts:
1. Hand-written alongside JavaScript source to type-check legacy code
2. Generated automatically by `tsc` with `"declaration": true` for published libraries
3. Published to `@types/*` packages by the DefinitelyTyped community

## Generating Declaration Files

### Automatic Generation

With `declaration: true` in `tsconfig.json`, the compiler generates a `.d.ts` file for every TypeScript source file:

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./types",
    "declarationMap": true
  }
}
```

`declarationMap` generates `.d.ts.map` files, enabling "go to definition" to navigate to the original TypeScript source rather than the generated declaration.

### What Gets Generated

Given this TypeScript source:

```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export class Vector {
  constructor(public x: number, public y: number) {}
  length(): number {
    return Math.sqrt(this.x ** 2 + this.y ** 2);
  }
}

export type Scalar = number | bigint;
```

The generated `math.d.ts` is:

```typescript
export declare function add(a: number, b: number): number;

export declare class Vector {
  x: number;
  y: number;
  constructor(x: number, y: number);
  length(): number;
}

export declare type Scalar = number | bigint;
```

## Writing Declaration Files by Hand

When adding types to an untyped JavaScript library, you write `.d.ts` files manually.

### Typing a CommonJS Module

```typescript
// @types/legacy-lib/index.d.ts

declare module "legacy-lib" {
  export interface Config {
    timeout?: number;
    retries?: number;
    baseUrl: string;
  }

  export interface Response<T = unknown> {
    data: T;
    status: number;
    headers: Record<string, string>;
  }

  export function request<T>(url: string, config?: Config): Promise<Response<T>>;
  export function setDefaults(config: Partial<Config>): void;

  export default {
    request,
    setDefaults,
  };
}
```

### Typing a Global Library (UMD/IIFE)

Libraries that attach to the global scope need ambient declarations:

```typescript
// globals.d.ts

declare const MY_LIBRARY_VERSION: string;

declare namespace MyLib {
  interface Options {
    theme: "light" | "dark";
    lang: string;
  }

  function init(options: Options): void;
  function destroy(): void;
}
```

### Wildcard Module Declarations

For file types that TypeScript does not understand natively (images, CSS, SVG, JSON with non-standard import):

```typescript
// declarations.d.ts

declare module "*.svg" {
  const content: string;
  export default content;
}

declare module "*.png" {
  const url: string;
  export default url;
}

declare module "*.module.css" {
  const styles: Record<string, string>;
  export default styles;
}

declare module "*.json" {
  const data: unknown;
  export default data;
}
```

## The `@types` Package System

TypeScript automatically loads type packages from `@types/*` in `node_modules`. When you install `@types/lodash`, TypeScript picks it up without any configuration change.

The resolution order for a module named `"foo"` is:
1. `node_modules/foo/index.d.ts`
2. `node_modules/foo/package.json` `"types"` or `"typings"` field
3. `node_modules/@types/foo/index.d.ts`

```bash
# Install types for a library
npm install --save-dev @types/express @types/node

# Check if a library has built-in types (look for "typings" in package.json)
cat node_modules/zod/package.json | grep typings
```

### `typeRoots` and `types`

Control which `@types` packages are loaded globally:

```json
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./custom-types"],
    "types": ["node", "jest"]
  }
}
```

With `types: [...]` set, only the listed `@types` packages are included as globals. Others can still be imported explicitly but won't add global types.

## Module Augmentation

Module augmentation adds new types to an existing module's declarations without modifying or replacing them.

### Augmenting a Third-Party Module

```typescript
// augmentations/express.d.ts

import "express"; // Ensure this is treated as a module augmentation

declare module "express" {
  interface Request {
    user?: {
      id: string;
      role: "admin" | "user";
    };
    correlationId?: string;
  }
}
```

After this augmentation, `req.user` and `req.correlationId` are recognized on `express.Request` throughout the project.

### Augmenting a Library's Generic Interfaces

A common pattern for type-safe environment variables, context stores, and plugin systems:

```typescript
// types/env.d.ts

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "production" | "test";
      DATABASE_URL: string;
      JWT_SECRET: string;
      PORT?: string;
    }
  }
}

export {}; // Ensure this file is a module, not a script
```

Now `process.env.NODE_ENV` is typed as `"development" | "production" | "test"` instead of `string | undefined`.

### React Component Props Augmentation

```typescript
// types/react.d.ts

import "react";

declare module "react" {
  interface HTMLAttributes<T> {
    "data-testid"?: string;
    "data-cy"?: string;
  }
}
```

This adds `data-testid` and `data-cy` as valid attributes on all HTML elements in JSX without TypeScript errors.

## Global Augmentation

Use `declare global {}` to extend the global scope from within a module:

```typescript
// lib/globals.ts

export {}; // Must be a module (has at least one import/export)

declare global {
  interface Window {
    analytics: {
      track(event: string, properties?: Record<string, unknown>): void;
      identify(userId: string, traits?: Record<string, unknown>): void;
    };
    __APP_CONFIG__: {
      apiUrl: string;
      featureFlags: Record<string, boolean>;
    };
  }

  interface Array<T> {
    // Extending a built-in interface
    groupBy<K extends string>(
      keyFn: (item: T) => K
    ): Record<K, T[]>;
  }
}
```

Including this file in the project (or importing it) makes `window.analytics` and `window.__APP_CONFIG__` accessible with full type safety.

## Interface Merging

When an interface is declared multiple times in the same scope, TypeScript merges all declarations. This is the mechanism behind module augmentation:

```typescript
interface Plugin {
  name: string;
  version: string;
}

// In another file or augmentation:
interface Plugin {
  initialize(config: unknown): void;
  teardown(): void;
}

// TypeScript sees the merged result:
// { name: string; version: string; initialize(...): void; teardown(): void }
```

Only interfaces support merging — not type aliases:

```typescript
type Foo = { a: string };
type Foo = { b: number }; // Error: Duplicate identifier 'Foo'
```

## Triple-Slash Directives

Triple-slash directives are XML-like comments that instruct the TypeScript compiler about dependencies:

```typescript
/// <reference types="node" />    // Include @types/node globally
/// <reference path="./globals.d.ts" /> // Include specific declaration file
/// <reference lib="es2022" />    // Include built-in lib declarations
```

These are mostly a legacy mechanism. Modern TypeScript projects use `tsconfig.json` `types` and `lib` fields instead. Triple-slash directives remain useful in `.d.ts` files published to `@types` to express dependencies between type packages.

## Publishing Library Types

When publishing a TypeScript library or a JavaScript library with types, configure `package.json`:

```json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    },
    "./utils": {
      "types": "./dist/utils.d.ts",
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.js"
    }
  }
}
```

The `types` (or `typings`) field points to the root declaration file. The `exports` map is required for subpath imports to resolve their types correctly in TypeScript 5+.

## Comparison Table

| Scenario | Mechanism |
|---------|-----------|
| Adding types to a JS library | Ambient module declaration in `.d.ts` |
| Global variables/functions | `declare const`, `declare function` in `.d.ts` |
| Extending an existing module's types | Module augmentation (`declare module "foo" {}`) |
| Extending global scope from a module | `declare global {}` |
| Extending built-in interfaces | Global augmentation or module augmentation |
| Non-JS file imports (SVG, CSS, etc.) | Wildcard module declaration (`declare module "*.svg"`) |
| Library publication | `declaration: true` + `types` field in `package.json` |

## Best Practices

### 1. Always Export an Empty Object in Global Augmentation Files

Without an import or export, a `.d.ts` file is treated as a script (contributing to the global scope automatically). With any export, it becomes a module, and you must use `declare global {}` for global modifications:

```typescript
// Without this export, the file contributes globally without declare global
// With it, you must explicitly opt in
export {};
```

### 2. Keep Augmentations in Dedicated Files

```
src/
  types/
    express.d.ts   // Express request augmentation
    env.d.ts       // process.env types
    window.d.ts    // window global augmentation
    assets.d.ts    // image/svg/css module declarations
```

### 3. Match Declaration Files to Source Files

When hand-writing types for a JavaScript module, mirror the module's structure precisely. Undeclared exports cause runtime surprises when TypeScript consumers expect types that don't exist.

### 4. Use `declare module` for Path-Mapped Modules

If your project uses path aliases, you may need declarations:

```typescript
// For "@/utils" -> "src/utils/index.ts"
// TypeScript resolves this via tsconfig.json paths — no declaration needed
// But for unresolvable aliases or virtual modules:

declare module "@/generated/api-client" {
  export * from "./real-location/api-client";
}
```

### 5. Prefer Built-in Types Over Manual Declarations for Known Environments

For Node.js, browser, and Web Worker environments, use `lib` and `types` in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["node"]
  }
}
```

## Interview Questions

### Q1: What is the difference between a `.d.ts` file and a TypeScript source file?

**Answer:** A `.d.ts` file contains only type declarations — interfaces, type aliases, function signatures, class declarations with `declare` — and no implementation code. It is never compiled to JavaScript; it only informs the TypeScript compiler about the types in a related JavaScript file or external library. A `.ts` source file contains both types and implementation and compiles to JavaScript. When you run `tsc` with `declaration: true`, it generates `.d.ts` files alongside `.js` output files, giving consumers of your library type information without the source.

### Q2: How does module augmentation work under the hood?

**Answer:** Module augmentation exploits TypeScript's interface merging. When you write `declare module "express" { interface Request { ... } }`, TypeScript merges your additions into the `Request` interface that express's original declaration file defines. This works because TypeScript loads all declaration files referenced in the project and merges same-named interfaces across files. The key requirement is that the augmentation must be in a module (contain at least one import/export), so TypeScript treats it as module augmentation rather than a redeclaration.

### Q3: How do you type a module that attaches to the global scope (UMD pattern)?

**Answer:** For UMD libraries (accessible both via `require` and as a global), TypeScript provides the `export as namespace` syntax in declaration files. You declare the module normally with `declare module "lib-name"` and also export it as a global namespace: `export as namespace LibName`. This tells TypeScript that when the library is loaded as a script (not via a module system), its exports are available as properties of the global `LibName` object. For purely global IIFE libraries, use `declare namespace` and `declare const` at the top level of a `.d.ts` file that is included (not imported) in the project.

### Q4: What is the purpose of the `types` field vs `typings` in `package.json`?

**Answer:** They are aliases — `types` and `typings` in `package.json` both point to the root declaration file of the package. TypeScript checks `types` first, then `typings`. Modern convention is to use `types`. When combined with `exports` (Node.js package exports map), TypeScript 5+ requires types to be declared in the `exports` map for each subpath as well, using the `"types"` condition. If neither `types` nor `typings` is set, TypeScript looks for `index.d.ts` in the package root or uses the `"main"` field with a `.d.ts` extension.

### Q5: When should you use `/// <reference types="..." />` vs `types` in `tsconfig.json`?

**Answer:** Use `/// <reference types="..." />` inside declaration files (`.d.ts`) that you publish, to express a dependency on another type package — for example, a declaration file that uses types from `@types/node` should include `/// <reference types="node" />` at the top. This ensures consumers of your package who don't have `@types/node` installed get a clear error. Use the `types` field in `tsconfig.json` for application-level projects to control which `@types` packages contribute global types, keeping the ambient environment predictable. Never use triple-slash references in regular `.ts` source files — configure the type resolution in `tsconfig.json` instead.

## Common Pitfalls

### 1. Forgetting `export {}` in Global Augmentation Files

```typescript
// WRONG: treated as a script, not a module
// declare global {} is not needed or recognized here
interface Window {
  myApp: App; // This works but also pollutes global scope unexpectedly
}

// CORRECT:
export {};
declare global {
  interface Window {
    myApp: App;
  }
}
```

### 2. Module Augmentation Only Works When the Original Module Is Loaded

```typescript
// Only effective if "express" is installed and used in the project
declare module "express" {
  interface Request {
    user?: User;
  }
}

// If express is not a dependency, this augmentation is effectively useless
```

### 3. Ambient Declarations Conflict with Import Declarations

```typescript
// Incorrect: mixing ambient and import in the same file
import type { User } from "./user";

declare module "some-lib" {
  // Cannot use imported types directly here without re-importing
  interface Config {
    onLoad: (user: User) => void; // May work, but depends on module resolution
  }
}
```

### 4. Over-Broad Wildcard Declarations

```typescript
// Too permissive: makes any string import valid
declare module "*" {
  const content: unknown;
  export default content;
}

// More specific:
declare module "*.svg" {
  const content: string;
  export default content;
}
```

### 5. Declaration Files in `src/` Are Not Automatically Type-Only

If you place a `.d.ts` file in `src/` with `import` statements, it becomes a regular module file and will participate in compilation. Use `declare module` syntax and ensure augmentation files use `export {}` to keep them purely declarative.

## Resources

### Official Documentation
- [TypeScript Handbook: Declaration Files](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html)
- [TypeScript Handbook: Module Augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation)
- [TypeScript Handbook: Declaration Reference](https://www.typescriptlang.org/docs/handbook/declaration-files/by-example.html)
- [Publishing TypeScript](https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html)

### Articles and Guides
- [DefinitelyTyped Contributing Guide](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/README.md)
- [TypeScript Deep Dive: Declaration Merging](https://basarat.gitbook.io/typescript/type-system/declaration-spaces)
- [How to Write Ambient Modules](https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html)

### Tools
- [dts-gen](https://github.com/microsoft/dts-gen): Microsoft tool for generating initial `.d.ts` files from JavaScript
- [TypeScript Playground](https://www.typescriptlang.org/play): Test declaration file patterns
- [arethetypeswrong.github.io](https://arethetypeswrong.github.io/): Check if a package's type exports are correctly configured
