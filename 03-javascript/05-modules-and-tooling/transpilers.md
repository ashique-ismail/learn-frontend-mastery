# Transpilers: Babel, SWC, and TSC

## Overview

A transpiler (source-to-source compiler) takes JavaScript or a JavaScript superset and produces JavaScript that targets a different environment — typically converting modern syntax to older equivalents, stripping type annotations, or transforming non-standard syntax like JSX into standard calls. The three dominant transpilers in the JavaScript ecosystem are Babel (the original, plugin-driven JS transpiler), SWC (a Rust-based Babel replacement with near-identical plugin semantics but dramatically faster execution), and TSC (the TypeScript compiler, which both type-checks and emits JavaScript).

Understanding when to use each, their performance characteristics, and their limitations is critical for engineers who own CI pipelines, configure build toolchains, or debug compilation issues in React, TypeScript, or monorepo projects.

## What Transpilers Do (and Don't Do)

### Core Responsibilities

```
Input:  TypeScript + modern JS + JSX
                   |
         [ Parse to AST ]
                   |
         [ Transform AST ]  <- where syntax transforms happen
                   |
         [ Code generation ] <- AST back to JS string
                   |
Output: JavaScript targeting specific environments
```

Transpilers handle:
- Stripping TypeScript type annotations
- Converting JSX to `React.createElement()` (or equivalent)
- Downleveling syntax: `async/await` → generators, optional chaining → ternaries
- Module format conversion: ESM → CJS (for Node.js compatibility)
- Stage 3+ proposal transforms

Transpilers do NOT handle:
- Polyfilling runtime APIs (e.g., `Array.prototype.at`, `Promise.allSettled`)
- Type checking (except TSC)
- Bundling
- Tree shaking

```javascript
// Transpiler transforms SYNTAX, not runtime APIs
const arr = [1, 2, 3];
const last = arr.at(-1); // NOT transformed — arr.at is a runtime API
                          // Requires a polyfill like core-js

const result = arr?.find?.(x => x > 1); // IS transformed — optional chaining syntax
// Becomes: var result = (_arr = arr) === null || _arr === void 0 ? void 0 : ...
```

## Babel

### Architecture

Babel processes code in three phases: **parse** (code → AST using `@babel/parser`), **transform** (plugins mutate the AST), and **generate** (`@babel/generator` emits code from the AST). Every transform is a plugin; preset packages bundle related plugins.

```bash
npm install --save-dev @babel/core @babel/cli @babel/preset-env @babel/preset-typescript @babel/preset-react
```

```javascript
// babel.config.js
module.exports = {
  presets: [
    [
      '@babel/preset-env',
      {
        targets: '> 0.5%, last 2 versions, not dead',
        // or explicit: targets: { chrome: '90', firefox: '88' }
        
        useBuiltIns: 'usage',  // inject polyfills where needed
        corejs: { version: 3, proposals: true },
        
        modules: false   // keep ESM (let bundler handle modules)
        // modules: 'commonjs'  // convert to CJS (for Node.js/Jest)
      }
    ],
    ['@babel/preset-typescript'],
    ['@babel/preset-react', { runtime: 'automatic' }]  // React 17+ JSX transform
  ],
  plugins: [
    '@babel/plugin-transform-class-properties',
    '@babel/plugin-proposal-decorators',          // stage 3
    ['babel-plugin-styled-components', { displayName: true }]
  ],
  env: {
    test: {
      presets: [['@babel/preset-env', { targets: { node: 'current' } }]]
    }
  }
};
```

### @babel/preset-env and Browserslist

`@babel/preset-env` is the meta-preset that selects plugins based on your target environments. It uses compat-data (which browsers support which features) combined with your `targets` setting to include only necessary transforms.

```javascript
// Only transforms features NOT supported by your targets
// If targets include Chrome 90+, you don't need optional chaining transform
// Because Chrome 90 supports it natively

// .browserslistrc (alternative to targets in babel config)
// Used by multiple tools: Autoprefixer, PostCSS, Babel, ESLint-plugin-compat
> 0.5%
last 2 versions
not dead
not IE 11
```

```bash
# Check what your browserslist resolves to
npx browserslist '> 0.5%, last 2 versions, not dead'
```

### useBuiltIns: Polyfill Strategy

```javascript
// useBuiltIns: false (default) — no polyfills injected
// You manage polyfills manually

// useBuiltIns: 'entry' — replaces `import 'core-js'` with specific polyfills
// based on your targets. Import once at app entry:
import 'core-js'; // replaced by Babel with ~100 specific polyfill imports

// useBuiltIns: 'usage' — Babel adds polyfills per-file based on usage
// No manual import needed — Babel instruments automatically
// Best option for most projects
```

```javascript
// With useBuiltIns: 'usage', this code:
const arr = [1, 2, 3];
arr.findIndex(x => x > 1);

// Gets compiled to (if target doesn't support findIndex):
require('core-js/modules/es.array.find-index.js');
const arr = [1, 2, 3];
arr.findIndex(x => x > 1);
```

### Babel Plugins: Writing a Transform

Understanding the plugin API reveals how Babel works internally:

```javascript
// A simple Babel plugin that transforms console.log to console.debug
module.exports = function({ types: t }) {
  return {
    name: 'console-log-to-debug',
    visitor: {
      CallExpression(path) {
        const { callee } = path.node;
        if (
          t.isMemberExpression(callee) &&
          t.isIdentifier(callee.object, { name: 'console' }) &&
          t.isIdentifier(callee.property, { name: 'log' })
        ) {
          callee.property.name = 'debug';
        }
      }
    }
  };
};
```

```javascript
// Plugins define visitors: AST node type → handler
// Babel traverses the AST and calls your visitor for each matching node
// path provides: node (AST node), parent, scope, context, replaceWith(), remove()
```

### Babel for Jest

```javascript
// jest.config.js — Babel for test transforms
module.exports = {
  transform: {
    '^.+\\.(js|jsx|ts|tsx)$': 'babel-jest'
  }
};

// babel.config.js env.test section handles CJS conversion for Jest
```

### Babel Limitations

- Written in JavaScript — slower than native alternatives (SWC, esbuild)
- Does not type-check TypeScript (transform-only, like a stripper)
- Some TypeScript features compile differently: `const enum` is unsupported, `namespace` merging has caveats
- Plugin quality varies widely

## SWC

### Architecture

SWC (Speedy Web Compiler) is a Rust-based transpiler created by Donny/강동윤. It is API-compatible with Babel for most common configurations and is 20-70x faster in benchmarks. It is used by Next.js (as the default compiler since v12), Vite's dev server, Parcel, and Deno.

```bash
npm install --save-dev @swc/core @swc/cli
# or for Jest:
npm install --save-dev @swc/jest
```

```json
// .swcrc
{
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "tsx": true,
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "react": {
        "runtime": "automatic",
        "development": false,
        "refresh": false
      }
    },
    "target": "es2020",
    "loose": false,
    "externalHelpers": false
  },
  "module": {
    "type": "commonjs"
  },
  "sourceMaps": true
}
```

```javascript
// @swc/core Node.js API
const { transform, transformFile } = require('@swc/core');

const output = await transform(sourceCode, {
  filename: 'input.ts',
  jsc: {
    parser: { syntax: 'typescript' },
    target: 'es2020'
  },
  module: { type: 'es6' }
});

console.log(output.code);    // transformed JS
console.log(output.map);     // source map
```

### SWC for Jest

SWC's most common use is as a Jest transform, replacing `babel-jest` for dramatically faster test runs:

```javascript
// jest.config.js
module.exports = {
  transform: {
    '^.+\\.(t|j)sx?$': [
      '@swc/jest',
      {
        jsc: {
          parser: { syntax: 'typescript', tsx: true },
          transform: { react: { runtime: 'automatic' } }
        }
      }
    ]
  }
};
```

```bash
# Typical benchmark results in large projects:
# babel-jest:  45 seconds
# @swc/jest:   8 seconds
# Same output, same semantics
```

### SWC in Next.js

Next.js 12+ uses SWC as the default compiler, replacing Babel:

```javascript
// next.config.js
module.exports = {
  swcMinify: true,    // use SWC for minification instead of Terser

  compiler: {
    // SWC equivalents of common Babel plugins
    styledComponents: true,
    removeConsole: { exclude: ['error'] },
    reactRemoveProperties: { properties: ['^data-test'] },
    emotion: true
  }
};
```

### SWC Plugin API

SWC has a Wasm-based plugin API for custom transforms, though it is less mature than Babel's:

```bash
npm install --save-dev @swc/plugin-styled-components
```

```json
{
  "jsc": {
    "experimental": {
      "plugins": [
        ["@swc/plugin-styled-components", { "displayName": true }]
      ]
    }
  }
}
```

### SWC Limitations

- Plugin ecosystem is smaller than Babel's (but growing)
- Wasm plugin API is less ergonomic than Babel's JS visitor API
- Some Babel plugins have no SWC equivalent yet
- Does not type-check (same as Babel — strip-only for TypeScript)

## TSC (TypeScript Compiler)

### Architecture

TSC is the official TypeScript compiler from Microsoft. Unlike Babel and SWC, it performs **full type checking** in addition to transpilation. It reads `tsconfig.json` for project configuration and processes the full TypeScript language including all type annotations, generics, declaration files, and project references.

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",  // for Vite/webpack projects
    // "moduleResolution": "Node16", // for Node.js projects
    
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    
    "jsx": "react-jsx",          // React 17+ transform
    // "jsx": "preserve",        // let Babel/SWC handle JSX
    
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,          // emit .d.ts files
    "declarationMap": true,       // source maps for .d.ts
    "sourceMap": true,
    
    "lib": ["ES2020", "DOM"],
    "skipLibCheck": true,         // skip type checking .d.ts files
    "esModuleInterop": true,      // default imports from CJS modules
    
    "incremental": true,          // cache compilation state in .tsbuildinfo
    "tsBuildInfoFile": "./dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### TSC as Type Checker Only (noEmit)

In most modern projects, TSC is used exclusively for type checking while Babel/SWC/esbuild handle transpilation. This decoupling improves build speed significantly.

```bash
# Type check only — no JavaScript emitted
npx tsc --noEmit

# Watch mode for development
npx tsc --noEmit --watch
```

```javascript
// vite.config.ts — Vite uses esbuild for transpilation, tsc separately for types
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc'; // uses SWC, not TSC

export default defineConfig({
  plugins: [react()]
  // TSC is NOT invoked during vite dev or vite build
  // Run tsc --noEmit separately in CI
});
```

```json
// package.json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "type-check": "tsc --noEmit",
    "test": "vitest"
  }
}
```

### Project References

For monorepos with multiple TypeScript packages, project references enable incremental compilation across packages:

```json
// tsconfig.json (root or composite package)
{
  "compilerOptions": {
    "composite": true,         // required for project references
    "declaration": true,       // required for composite
    "declarationMap": true
  },
  "references": [
    { "path": "../../packages/ui" },
    { "path": "../../packages/utils" }
  ]
}
```

```bash
# Build all referenced projects in dependency order
npx tsc --build

# Build and watch all references
npx tsc --build --watch

# Clean build artifacts
npx tsc --build --clean
```

### Declaration Files (.d.ts)

TSC emits `.d.ts` declaration files that allow TypeScript consumers of your library to get full type information:

```bash
# For a library, emit both JS and .d.ts
tsc --declaration --declarationMap --outDir dist

# dist/
#   index.js         <- transpiled JS
#   index.d.ts       <- type declarations
#   index.d.ts.map   <- source map linking .d.ts back to .ts
```

```json
// package.json for a TypeScript library
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

### TSC and moduleResolution

The `moduleResolution` setting is critical and frequently misconfigured:

```json
{
  "compilerOptions": {
    // "node"        — pre-ESM Node.js resolution (CJS only)
    // "node16"      — Node.js 16+ ESM/CJS resolution (requires .js extensions in imports)
    // "nodenext"    — same as node16 but tracks Node.js updates
    // "bundler"     — for apps using Vite/webpack (no extension requirements, supports package.json exports)
    
    "moduleResolution": "Bundler"  // correct for Vite/webpack apps
  }
}
```

```typescript
// With "moduleResolution": "Node16", relative imports need .js extensions
// even though the source files are .ts (TypeScript transpiles .ts → .js)
import { helper } from './utils.js'; // correct for Node16
import { helper } from './utils';    // error with Node16, OK with Bundler
```

## Comparison Table

| Feature | Babel | SWC | TSC |
|---|---|---|---|
| Language | JavaScript | Rust | TypeScript |
| Speed | Baseline | 20-70x faster than Babel | Slow for large projects |
| Type checking | No (strip only) | No (strip only) | Yes |
| Plugin ecosystem | Huge (JS plugins) | Growing (Wasm plugins) | Limited transforms |
| JSX support | Yes | Yes | Yes |
| TypeScript support | Strip only | Strip only | Full |
| Polyfill injection | Yes (via preset-env) | No | No |
| Declaration files | No | No | Yes |
| Incremental builds | Limited | No (fast enough) | Yes (tsbuildinfo) |
| Project references | No | No | Yes |
| Primary use | Legacy/complex transforms | Fast transforms in tools | Type checking + .d.ts |
| Used by | Legacy apps, CRA | Next.js, Vite | All TS projects for types |

## Best Practices

### 1. Separate Type Checking from Transpilation

```json
{
  "scripts": {
    "build": "tsc --noEmit && vite build",
    "dev": "vite",
    "type-check": "tsc --noEmit"
  }
}
```

Let esbuild/SWC handle the fast per-file transforms. Run TSC only for type checking.

### 2. Use SWC for Jest Instead of Babel

```bash
npm install --save-dev @swc/jest
```

The performance difference is significant for projects with hundreds of test files. The semantic output is equivalent to babel-jest.

### 3. Target Environments Explicitly

```javascript
// Babel
targets: { browsers: '> 0.5%, last 2 versions, not dead' }
// or for Node.js only:
targets: { node: '20' }

// TSC
"target": "ES2022",  // match what your runtime actually supports
```

Avoid targeting ES5 unless you truly need IE11 support. Each downlevel transform adds bundle size and reduces debuggability.

### 4. Use runtime: 'automatic' for React JSX

```json
// Babel
["@babel/preset-react", { "runtime": "automatic" }]

// SWC
{ "jsc": { "transform": { "react": { "runtime": "automatic" } } } }

// TSC
{ "jsx": "react-jsx" }
```

The automatic runtime eliminates the need for `import React from 'react'` in every file and produces slightly smaller output.

### 5. Keep Babel Config in babel.config.js Not .babelrc

```javascript
// babel.config.js — applies to all files including node_modules (for monorepos)
// .babelrc — applies only to files relative to that directory
// Use babel.config.js for projects with workspaces or Jest
```

## Interview Questions

### Q1: What is the difference between a transpiler and a polyfill?

**Answer:** A transpiler is a source-to-source compiler that transforms syntax. It converts `async/await` to `Promise`-based code, optional chaining to nested ternaries, or TypeScript to JavaScript. The output syntax expresses equivalent logic using only constructs the target environment understands. A polyfill provides a runtime implementation of an API that does not exist in the target environment. `Array.prototype.at`, `Promise.allSettled`, `globalThis`, and `structuredClone` are APIs that must be polyfilled — no syntax transform can make `Array.prototype.at` work if the environment's `Array.prototype` does not have an `at` method. Babel with `core-js` handles both: `@babel/preset-env` transforms syntax and `useBuiltIns: 'usage'` injects polyfills.

### Q2: Why does TypeScript compilation in Vite not report type errors?

**Answer:** Vite uses esbuild to transform TypeScript files. esbuild strip-compiles TypeScript — it removes type annotations and converts TypeScript syntax to JavaScript but performs no type analysis. This is intentional: esbuild is a per-file transform without cross-file type information, and type checking would require the full TypeScript language service with access to all source files, `.d.ts` declarations, and `tsconfig.json`. Vite documents this explicitly. To get type errors, you must run `tsc --noEmit` separately — typically as a pre-build step (`"build": "tsc --noEmit && vite build"`) or in a separate CI job. Some editors (VS Code with the TypeScript Language Server) provide type errors inline during development independently of Vite.

### Q3: When would you choose Babel over SWC?

**Answer:** Choose Babel when you need a custom transform that has no SWC equivalent. Babel's JavaScript plugin API is mature, well-documented, and has thousands of community plugins — for macros (`babel-plugin-macros`), styled-components display names, module remapping, testing utilities, and more. If your project uses a Babel plugin that SWC does not support via its Wasm plugin API, you must use Babel. Also choose Babel for projects that use advanced `preset-env` polyfill injection with `core-js`, as SWC does not have equivalent polyfill injection. For straightforward TS/JSX transforms, SWC is the better choice due to speed. Many projects use esbuild or SWC for transpilation and keep a minimal Babel config for specific plugins only.

### Q4: What are TypeScript project references and when do you use them?

**Answer:** Project references allow a TypeScript project to declare dependencies on other TypeScript projects. When you run `tsc --build`, TypeScript compiles referenced projects in dependency order and caches their outputs (`tsbuildinfo`). Only projects whose inputs have changed are recompiled. This enables incremental compilation in monorepos where rebuilding the full project graph on every change would be prohibitively slow. To use project references, each referenced package must set `"composite": true` and emit declaration files. The referencing `tsconfig.json` lists paths in the `"references"` array. The tradeoff is that you must keep `tsconfig.json` references in sync with actual code dependencies — a divergence causes stale type information.

### Q5: What is the difference between `module` and `moduleResolution` in tsconfig?

**Answer:** `"module"` controls the module syntax that TypeScript emits in output files: `CommonJS` outputs `require()`/`module.exports`, `ESNext` outputs `import`/`export`, etc. `"moduleResolution"` controls the algorithm TypeScript uses to resolve import specifiers to file paths during type checking. They are separate concerns. For example, `"module": "CommonJS"` with `"moduleResolution": "Bundler"` means: type-check using bundler resolution rules (which understand package.json `exports` and don't require `.js` extensions) but emit CJS output. A common mistake is setting `"module": "ESNext"` without a compatible `moduleResolution` — pre-Bundler resolution modes may fail to find packages that only expose their ESM build via the `exports` field.

## Common Pitfalls

### 1. Using TSC for Speed-Critical Transforms

```bash
# Don't use tsc for per-file transforms in CI — it's slow for large projects
# Do this:
tsc --noEmit                  # type check only (parallel to tests)
esbuild / swc                  # fast transpilation

# Not this (in hot paths):
tsc --outDir dist              # slow — full type check + emit
```

### 2. Misunderstanding Babel's TypeScript Support

```typescript
// This is VALID TypeScript syntax
const enum Direction { Up, Down }
// Compiles to: var Direction; (enum value is inlined)

// Babel with @babel/preset-typescript CANNOT compile const enum correctly
// Babel operates file-by-file without cross-file knowledge
// Use isolatedModules: true in tsconfig to catch these incompatibilities:
```

```json
{
  "compilerOptions": {
    "isolatedModules": true  // errors on features Babel/SWC can't handle
  }
}
```

### 3. Targeting ES5 Unnecessarily

```javascript
// Targeting ES5 generates significantly larger output due to:
// - async/await → complex generator polyfills
// - classes → prototype chains with helper functions
// - arrow functions → regular functions with `this` binding

// Unless you need IE11, target ES2017+ (async/await native)
targets: { browsers: '> 0.5%, last 2 versions, not IE 11' }
```

### 4. Forgetting to Run Type Checks in CI

```yaml
# GitHub Actions — run type check alongside tests
- name: Type check
  run: npx tsc --noEmit

- name: Test
  run: npm test
```

Skipping `tsc --noEmit` in CI means type errors only surface in editors, not in the merge process.

### 5. Conflating Babel Presets and Plugins

```javascript
// Presets are ordered RIGHT TO LEFT
presets: ['@babel/preset-env', '@babel/preset-typescript', '@babel/preset-react']
// Execution order: react → typescript → env
// JSX must be transformed to JS before preset-env processes it

// Plugins are ordered LEFT TO RIGHT (opposite of presets)
plugins: ['transform-decorators', 'transform-class-properties']
// Execution order: decorators → class-properties
```

## Resources

### Official Documentation
- [Babel Documentation](https://babeljs.io/docs/)
- [SWC Documentation](https://swc.rs/docs/getting-started)
- [TypeScript Compiler Options](https://www.typescriptlang.org/tsconfig)
- [TSC Project References](https://www.typescriptlang.org/docs/handbook/project-references.html)

### Articles and Guides
- [SWC vs Babel: Performance Comparison](https://swc.rs/blog/perf-swc-vs-babel)
- [TypeScript's isolatedModules flag explained](https://www.typescriptlang.org/tsconfig#isolatedModules)
- [How Babel works (plugin handbook)](https://github.com/jamiebuilds/babel-handbook)

### Tools
- [babel-plugin-tester](https://github.com/babel-utils/babel-plugin-tester): Test Babel plugins
- [AST Explorer](https://astexplorer.net/): Inspect AST output for Babel/SWC
- [tsup](https://tsup.egoist.dev/): esbuild-based TS library build tool
- [ts-node](https://typestrong.org/ts-node/): Run TypeScript directly in Node.js
