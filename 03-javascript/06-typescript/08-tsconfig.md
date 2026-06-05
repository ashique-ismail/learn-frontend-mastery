# `tsconfig.json` Essentials

## The Idea

**In plain English:** `tsconfig.json` is a settings file that tells TypeScript how to do its job on your project — which files to check, how picky to be about mistakes, and what kind of JavaScript to produce as output. Think of it as a control panel where you adjust all the rules TypeScript follows.

**Real-world analogy:** Imagine setting up a spell-checker and grammar tool in a word processor before writing an essay. You choose the language (English vs. British English), how strict the grammar rules are, which folders of documents to scan, and where to save the finished, corrected copy.
- The language setting (English vs. British English) = `target` (which version of JavaScript to output)
- The strictness level (casual vs. academic grammar) = `strict` (how hard TypeScript checks your code for errors)
- The folders to scan = `include` / `exclude` (which files TypeScript looks at)
- The destination folder for the finished copy = `outDir` (where the compiled JavaScript files are saved)

---

## Overview

`tsconfig.json` is the configuration file that tells the TypeScript compiler how to process your project — which files to include, which JavaScript features to target, how strictly to check types, and how to handle modules. Understanding `tsconfig.json` deeply is essential for debugging build failures, optimizing compile performance, and ensuring your TypeScript code produces JavaScript that works correctly in its target environment.

## Basic Structure

```json
{
  "compilerOptions": {},
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"],
  "references": [],
  "extends": "./tsconfig.base.json"
}
```

- `compilerOptions` — compiler behavior settings (the main section)
- `include` — glob patterns for files to include in the compilation
- `exclude` — glob patterns for files to exclude (defaults to `node_modules`, `outDir`, `declarationDir`)
- `files` — explicit list of files to compile (overrides `include`/`exclude` for specific files)
- `references` — project references for monorepo setups
- `extends` — inherits settings from another `tsconfig.json`

## Target and Language Level

### `target`

Specifies the ECMAScript version that the compiler outputs. TypeScript will downcompile syntax and inject polyfills via helpers to match the target:

```json
{
  "compilerOptions": {
    "target": "ES2022"
  }
}
```

Common targets: `"ES5"`, `"ES6"` / `"ES2015"`, `"ES2020"`, `"ES2022"`, `"ESNext"`.

Setting `target` affects:
- Syntax downcompilation (arrow functions, async/await, destructuring, etc.)
- Which built-in types are available (based on the `lib` default)
- Whether TypeScript emits helpers directly or expects them from a polyfill

### `lib`

Specifies which built-in type declarations to include. Defaults are based on `target`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"]
  }
}
```

Key lib groups:
- `"ES2022"` — ES2022 language features (includes ES2021, ES2020, etc.)
- `"DOM"` — browser APIs (`document`, `window`, `fetch`, etc.)
- `"DOM.Iterable"` — iterable DOM collections (NodeList, HTMLCollection)
- `"WebWorker"` — Service Worker and Web Worker APIs
- `"ESNext"` — latest unstable features

For Node.js projects, use `"lib": ["ES2022"]` without `"DOM"` and install `@types/node`.

### `useDefineForClassFields`

Determines whether class fields use `Object.defineProperty` (ECMAScript semantics) or direct assignment:

```json
{
  "compilerOptions": {
    "useDefineForClassFields": true
  }
}
```

Defaults to `true` when `target` is `"ES2022"` or higher. Affects how decorators interact with class fields.

## Module System

### `module`

Specifies the module system used in output JavaScript:

```json
{
  "compilerOptions": {
    "module": "NodeNext"
  }
}
```

Common values:
- `"CommonJS"` — `require`/`module.exports` (classic Node.js)
- `"ES2022"` / `"ESNext"` — `import`/`export` (browser ESM, bundlers)
- `"NodeNext"` / `"Node16"` — Node.js ESM with full `package.json` `"exports"` resolution
- `"Preserve"` — keep whatever module syntax the source uses (TypeScript 5.4+)

`"NodeNext"` is the recommended setting for modern Node.js projects because it enforces `.js` extensions on relative imports and fully implements the Node.js module resolution algorithm.

### `moduleResolution`

Controls how TypeScript resolves module imports:

```json
{
  "compilerOptions": {
    "moduleResolution": "Bundler"
  }
}
```

Common values:
- `"Node10"` / `"Node"` (legacy) — Node.js CommonJS resolution, tries many file extensions
- `"Node16"` / `"NodeNext"` — matches Node.js ESM/CJS resolution, must match `module`
- `"Bundler"` — for bundlers (Webpack, Vite, esbuild) that handle resolution themselves; allows extensionless imports

For applications using a bundler, `"Bundler"` is the correct choice. For Node.js without a bundler, use `"NodeNext"`.

### `paths`

Maps import paths to file system locations (path aliases):

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@utils/*": ["./src/utils/*"]
    }
  }
}
```

`paths` is purely a TypeScript type-resolution hint — it does not configure the module bundler. You must also configure matching aliases in Webpack, Vite, or your bundler of choice.

### `baseUrl`

Sets the base directory for non-relative module resolution. Required when using `paths`:

```json
{
  "compilerOptions": {
    "baseUrl": "./src"
  }
}
```

### `esModuleInterop`

Enables compatibility shims for importing CommonJS modules with ES module `import` syntax:

```json
{
  "compilerOptions": {
    "esModuleInterop": true
  }
}
```

Without `esModuleInterop`, importing a CJS module requires a namespace import:
```typescript
import * as fs from "fs"; // without esModuleInterop
import fs from "fs";      // with esModuleInterop — default import allowed
```

### `allowSyntheticDefaultImports`

Enabled automatically by `esModuleInterop`. Allows `import X from "module"` when the module has no default export, without runtime changes. Useful for bundler environments that handle interop themselves.

## Strict Mode Options

### `strict`

A shorthand that enables a bundle of strictness flags simultaneously:

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

`strict: true` enables:
- `noImplicitAny` — error when TypeScript infers `any`
- `strictNullChecks` — `null` and `undefined` are distinct types
- `strictFunctionTypes` — stricter function parameter variance
- `strictBindCallApply` — typed `bind`, `call`, `apply` methods
- `strictPropertyInitialization` — class properties must be initialized
- `noImplicitThis` — error when `this` has type `any`
- `alwaysStrict` — emits `"use strict"` in output and parses in strict mode
- `useUnknownInCatchVariables` — catch clause variables are `unknown`, not `any`

All new TypeScript projects should start with `strict: true`. Disabling individual flags is acceptable for legacy migration.

### `strictNullChecks`

The most impactful strict flag. When enabled, `null` and `undefined` are only assignable to their own types (and `any`/`unknown`):

```typescript
// strictNullChecks: false (off)
const name: string = null; // OK

// strictNullChecks: true
const name: string = null; // Error: Type 'null' is not assignable to type 'string'
const name: string | null = null; // OK
```

### `noUncheckedIndexedAccess`

Adds `undefined` to the result type of indexed access on arrays and objects with index signatures:

```json
{
  "compilerOptions": {
    "noUncheckedIndexedAccess": true
  }
}
```

```typescript
const arr: string[] = ["a", "b"];
const first = arr[0]; // string | undefined (with noUncheckedIndexedAccess)
// Without this flag: string

const map: Record<string, number> = {};
const val = map["key"]; // number | undefined (with flag)
```

This is not included in `strict: true` but is highly recommended. It catches a large class of off-by-one and missing-key bugs.

### `exactOptionalPropertyTypes`

Distinguishes between a property being absent and a property being `undefined`:

```json
{
  "compilerOptions": {
    "exactOptionalPropertyTypes": true
  }
}
```

```typescript
interface Config {
  timeout?: number;
}

// With exactOptionalPropertyTypes: true
const c: Config = { timeout: undefined }; // Error: cannot set optional property to undefined
const c2: Config = {};                    // OK — absent is different from undefined
```

## Output Options

### `outDir` and `rootDir`

```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

`rootDir` — the root of the source files (maintains directory structure in output).
`outDir` — where compiled `.js` files go.

### `declaration` and `declarationDir`

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./dist/types",
    "declarationMap": true
  }
}
```

For published packages, always emit declaration files. `declarationMap` enables "go to source" navigation.

### `sourceMap`

Generates `.js.map` files for debugger navigation:

```json
{
  "compilerOptions": {
    "sourceMap": true,
    "inlineSources": true
  }
}
```

`inlineSources` embeds the original TypeScript source inside the source map, enabling debugging without distributing the `.ts` files separately.

### `noEmit`

When TypeScript is used only for type checking and a bundler handles compilation:

```json
{
  "compilerOptions": {
    "noEmit": true
  }
}
```

This is common in projects using Vite, esbuild, or SWC where TypeScript is not the transpiler.

### `removeComments`

```json
{
  "compilerOptions": {
    "removeComments": true
  }
}
```

Strips all comments from output. Useful for production builds to reduce file size.

## Type Checking Behavior

### `skipLibCheck`

Skips type checking of all declaration files (`.d.ts`):

```json
{
  "compilerOptions": {
    "skipLibCheck": true
  }
}
```

Nearly all real-world projects enable this because conflicts between `@types` packages would otherwise cause spurious errors. The downside is that genuine errors in your own declaration files are also skipped.

### `isolatedModules`

Requires each file to be compilable independently, without type information from other files:

```json
{
  "compilerOptions": {
    "isolatedModules": true
  }
}
```

This is required by transpilers like esbuild and SWC that process one file at a time. It forbids:
- `const enum` (requires cross-file type information)
- Re-exporting types without `export type`
- `namespace` merging across files

### `verbatimModuleSyntax`

TypeScript 5.0+. Enforces that `import type` is used for type-only imports, and that they are erased from output. Replaces `isolatedModules` + `importsNotUsedAsValues`:

```json
{
  "compilerOptions": "verbatimModuleSyntax": true
}
```

```typescript
// Required with verbatimModuleSyntax
import type { User } from "./user"; // OK — will be erased
import { processUser } from "./user"; // OK — value import
import { User } from "./user"; // Error if User is only used as a type
```

### `noEmitOnError`

Prevents outputting JavaScript if there are any type errors:

```json
{
  "compilerOptions": {
    "noEmitOnError": true
  }
}
```

Important for CI pipelines and build systems where you want type errors to block deployment.

## Project Configuration Patterns

### Base Configuration

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "target": "ES2022",
    "lib": ["ES2022"],
    "moduleResolution": "Bundler",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

### Node.js Application

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

### Browser/Bundler Application (React, Vite)

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "noEmit": true,
    "verbatimModuleSyntax": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"]
}
```

### Library Package

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2020",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationDir": "./dist",
    "declarationMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

### Monorepo with Project References

```json
// packages/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "composite": true
  },
  "references": [
    { "path": "../shared" }
  ]
}
```

`composite: true` enables project references, which allow incremental builds — TypeScript only recompiles changed packages.

## Comparison Table

| Option | Default | Recommended | Impact |
|--------|---------|-------------|--------|
| `strict` | false | `true` | Enables all strict checks |
| `strictNullChecks` | false | `true` (via `strict`) | Prevents null/undefined bugs |
| `noUncheckedIndexedAccess` | false | `true` | Catches missing array/map entries |
| `exactOptionalPropertyTypes` | false | `true` | Distinguishes absent vs undefined |
| `skipLibCheck` | false | `true` | Avoids @types package conflicts |
| `esModuleInterop` | false | `true` | Cleaner CJS default imports |
| `isolatedModules` | false | `true` (bundler projects) | Enables esbuild/SWC compatibility |
| `verbatimModuleSyntax` | false | `true` | Enforces explicit type imports |
| `noEmit` | false | `true` (bundler projects) | Type-check only, no output |
| `declaration` | false | `true` (libraries) | Emit `.d.ts` files |

## Best Practices

### 1. Always Start with `strict: true`

No new project should be started without `strict: true`. It catches the majority of common bugs at compile time and prevents `null`/`undefined` errors from reaching production.

### 2. Use `extends` for Shared Configuration

In a monorepo, define a root `tsconfig.base.json` with shared settings and extend it in each package. This avoids drift between package configurations.

### 3. Separate Type-Check and Build Configs

For bundler projects, have a `tsconfig.json` for your editor/type-checking (with full paths, includes, etc.) and a separate `tsconfig.build.json` for building that excludes test files:

```json
// tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "**/__tests__/**"]
}
```

### 4. Use `NodeNext` or `Bundler` Module Resolution, Not Legacy `Node`

The legacy `"Node"` / `"Node10"` resolution does not handle the Node.js `exports` map in `package.json`, which can lead to import resolution mismatches. Always specify a modern resolution mode that matches your runtime.

### 5. Enable `noUncheckedIndexedAccess`

This is not included in `strict: true` but catches a large class of real bugs:

```typescript
const items = getItems();
const first = items[0]; // string | undefined with flag, string without
first.toUpperCase(); // Potential crash without the check
```

## Interview Questions

### Q1: What is the difference between `module` and `moduleResolution` in `tsconfig.json`?

**Answer:** `module` specifies the module system format used in the emitted JavaScript output — whether TypeScript outputs `require`/`module.exports` (CommonJS), `import`/`export` (ESM), or something else. `moduleResolution` specifies how TypeScript resolves `import` statements to file system paths during type checking. They are related but independent: `module: "CommonJS"` outputs CJS, but the resolution algorithm is configured by `moduleResolution`. For modern Node.js, both should be set to `"NodeNext"` because Node's resolution algorithm differs significantly from older bundler-based resolution. For bundler projects, `module: "ESNext"` with `moduleResolution: "Bundler"` is the correct pairing.

### Q2: What does `strict: true` enable and why should you always use it?

**Answer:** `strict: true` enables a bundle of strict checks including `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`, and `useUnknownInCatchVariables`. Most importantly, `strictNullChecks` makes `null` and `undefined` distinct types, preventing the majority of null-pointer-style runtime errors. Without `strict: true`, TypeScript's type checking is lenient to the point of missing major categories of bugs. Every new project should use `strict: true` from the beginning; retrofitting it onto an existing codebase is painful but worthwhile.

### Q3: What is `skipLibCheck` and what are the tradeoffs?

**Answer:** `skipLibCheck: true` skips type checking of all `.d.ts` files, including those in `node_modules/@types`. It is nearly universally needed in practice because different `@types` packages sometimes have conflicting type definitions — for example, two packages that each define global types for `Promise` or DOM APIs in incompatible ways. Without `skipLibCheck`, these conflicts would block compilation. The tradeoff is that genuine errors in your own declaration files are also skipped. The pragmatic recommendation is to enable it in most projects and carefully review any hand-written `.d.ts` files with other means.

### Q4: What is `isolatedModules` and why does it matter for bundlers like esbuild?

**Answer:** Bundlers like esbuild and SWC process TypeScript files one at a time, without the cross-file type information that `tsc` accumulates. This means they cannot safely erase type-only constructs that require cross-file resolution. `isolatedModules: true` enforces that your code is compatible with this single-file processing mode. It forbids `const enum` (requires inlining across files), requires `export type` for type re-exports, and disallows `namespace` merging. The newer `verbatimModuleSyntax` is the recommended replacement in TypeScript 5+, as it also enforces `import type` usage explicitly.

### Q5: When would you use project references (`references`) in `tsconfig.json`?

**Answer:** Project references are used in monorepos where multiple TypeScript packages depend on each other. With `composite: true` and `references`, TypeScript can perform incremental builds — only recompiling packages that have changed and their dependents. Without references, `tsc --build` compiles everything every time, which is slow in large repos. References also enforce correct dependency ordering and prevent circular dependencies at the build level. Each referenced project must have `composite: true`, `declaration: true`, and an explicit `outDir`. Project references are essential infrastructure for large TypeScript monorepos alongside tools like Turborepo.

## Common Pitfalls

### 1. Mismatching `module` and `moduleResolution`

```json
// WRONG: module and moduleResolution must be compatible
{
  "module": "CommonJS",
  "moduleResolution": "NodeNext" // Error: NodeNext requires module: NodeNext/Node16
}

// CORRECT combinations:
// { "module": "NodeNext", "moduleResolution": "NodeNext" } — Node.js ESM
// { "module": "CommonJS", "moduleResolution": "Node10" } — classic Node.js
// { "module": "ESNext", "moduleResolution": "Bundler" } — bundler project
```

### 2. `paths` Does Not Configure the Bundler

```json
// TypeScript knows about this alias, but Webpack/Vite does not
{
  "paths": { "@/*": ["./src/*"] }
}

// You must also configure Vite:
// vite.config.ts: resolve: { alias: { "@": path.resolve(__dirname, "src") } }
```

### 3. `include` Does Not Override `exclude`

If a path matches both `include` and `exclude`, `exclude` wins. Files in `node_modules` are excluded by default even if `include` covers them.

### 4. Forgetting `rootDir` Causes Extra Output Directories

Without `rootDir`, TypeScript computes the common root of all included files. If you have files in both `src/` and `test/`, TypeScript may output to `dist/src/` and `dist/test/` instead of `dist/`. Setting `rootDir: "./src"` prevents test files from affecting the output structure.

### 5. `noEmit` and `declaration` Are Mutually Exclusive

```json
// These are contradictory:
{
  "noEmit": true,
  "declaration": true
}

// With noEmit, no files are output — including .d.ts files
// Use separate tsconfig.json files for type-checking and building
```

## Resources

### Official Documentation
- [TypeScript Handbook: tsconfig.json Reference](https://www.typescriptlang.org/tsconfig)
- [TypeScript Handbook: Project References](https://www.typescriptlang.org/docs/handbook/project-references.html)
- [TypeScript Module Resolution Guide](https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html)

### Articles and Guides
- [Total TypeScript: TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet)
- [Are the Types Wrong?](https://arethetypeswrong.github.io/): Diagnose package.json exports/types issues
- [TypeScript Strict Mode Breakdown](https://mariusschulz.com/blog/strict-type-checking-in-typescript)

### Tools
- [tsconfig/bases](https://github.com/tsconfig/bases): Recommended tsconfig base files for common environments (Node LTS, React, Remix, etc.)
- [TypeScript Playground Config](https://www.typescriptlang.org/play): Experiment with compiler options interactively
