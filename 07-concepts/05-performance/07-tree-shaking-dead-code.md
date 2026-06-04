# Tree Shaking and Dead Code Elimination

## Overview

Tree shaking is the bundler process of statically analyzing your import graph and removing exports that are never used. The name comes from the metaphor of shaking a dependency tree — dead branches fall off. When it works, tree shaking is one of the most impactful ways to reduce JavaScript bundle size. When it doesn't work (and there are many ways it silently fails), you ship kilobytes of code users never execute. This guide explains the prerequisites, how bundlers implement it, and why it so frequently fails silently.

## Why Tree Shaking Requires ESM

Tree shaking is only possible with ES Modules (`import`/`export`) — not CommonJS (`require`/`module.exports`).

### CommonJS: Dynamic, Not Analyzable

```javascript
// CommonJS — requires evaluated at runtime, not statically analyzable
const utils = require('./utils');

// Which exports are used? Unknowable until runtime:
const method = process.env.FEATURE ? 'methodA' : 'methodB';
utils[method]();  // bundler can't statically determine this

// Also dynamic:
if (someCondition) {
  const helpers = require('./helpers');
}
```

CommonJS `require()` is a function call — it can be conditional, it can use a string variable, it can be called anywhere in any scope. Bundlers cannot safely remove any export because they can't prove it's unused.

### ES Modules: Static, Analyzable

```javascript
// ES Modules — import bindings are static, declared at top level
import { formatDate, parseDate } from './date-utils';
// The bundler knows EXACTLY which exports are used: formatDate and parseDate
// → parseDate unused? Remove it.
```

ESM imports:
- Must be at the top level (not inside functions or conditionals)
- Must use string literals (not variables)
- Are declarations, not expressions

This static structure lets bundlers build a complete import graph at build time.

## The Import Graph and Dead Code

```
Dependency graph — only used code survives:

app.js
 ├─ import { Button } from './ui'
 │     ui.js exports: Button, Modal, Tooltip, Dropdown
 │     Only Button imported → Modal, Tooltip, Dropdown eliminated
 │
 ├─ import { formatDate } from './date-utils'
 │     date-utils.js exports: formatDate, parseDate, formatTime
 │     Only formatDate imported → parseDate, formatTime eliminated
 │
 └─ import './analytics'
       analytics.js has side effects (modifies globals)
       Even if nothing is imported, it is INCLUDED
       (side effect — can't safely remove)
```

## sideEffects in package.json

The `sideEffects` field tells bundlers whether a package's modules are safe to tree-shake. This is the single most common reason tree shaking silently fails.

### What Is a Side Effect?

A module has a side effect if importing it changes global state, even if none of its exports are used:

```javascript
// side-effect.js — modifies globals on import
Array.prototype.unique = function() { /* ... */ };        // polyfill
window.__ANALYTICS__ = initAnalytics();                   // global init
document.addEventListener('click', globalClickHandler);   // listener
```

### package.json sideEffects Declaration

```json
// package.json

// "All files in this package are side-effect free — tree-shake freely"
{
  "sideEffects": false
}

// "Only these files have side effects — tree-shake everything else"
{
  "sideEffects": [
    "./src/polyfills.js",
    "./src/global-styles.css",
    "**/*.css"
  ]
}

// "All files may have side effects" (default if field is missing)
// (no sideEffects field) → bundler assumes every module is impure → no tree shaking
```

### Why Missing sideEffects Breaks Tree Shaking

```
Without sideEffects: false:

import { Button } from 'my-ui-library';

Bundler thinks:
  "my-ui-library/index.js might have side effects when imported.
   I must include the entire file, even if Button is the only export used."

→ Entire library shipped, not just Button

With sideEffects: false:

Bundler knows:
  "Safe to only include the parts that are actually imported."
→ Only Button's code + its dependencies shipped
```

### Checking if a Library Is Tree-Shakeable

```bash
# Check if the package declares sideEffects
cat node_modules/lodash-es/package.json | grep sideEffects
# sideEffects: false → good

cat node_modules/lodash/package.json | grep sideEffects
# (nothing) → CommonJS — not tree-shakeable; use lodash-es instead
```

## Bundler Implementation

### Webpack Tree Shaking

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',            // enables tree shaking automatically
  optimization: {
    usedExports: true,           // marks unused exports
    minimize: true,              // minifier removes marked code
    concatenateModules: true,    // scope hoisting — merges modules for better elimination
  },
};

// In development, use:
optimization: {
  usedExports: true,             // mark unused — shows /* unused harmony export */
  minimize: false,               // don't minify for readability
}
```

Webpack marks unused exports with comments, then the minifier (Terser) removes the dead code. Without `minimize: true`, the dead code stays in the bundle (just marked).

### Vite / Rollup Tree Shaking

Rollup (which Vite uses for production builds) has more aggressive tree shaking than Webpack:

```javascript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      treeshake: {
        moduleSideEffects: false,    // treat all modules as side-effect free
        propertyReadSideEffects: false,
        tryCatchDeoptimization: false, // don't bail out on try/catch
      },
    },
  },
});
```

Rollup performs "scope hoisting" by default — it inlines all modules into one flat scope, making dead code elimination easier and the output smaller. Webpack added this as "Module Concatenation Plugin."

## Common Tree-Shaking Failures

### 1. Named Re-Exports from Barrel Files

```typescript
// ❌ Barrel file that re-exports everything
// components/index.ts
export { Button } from './Button';
export { Modal } from './Modal';
export { Tooltip } from './Tooltip';
export { Dropdown } from './Dropdown';
export { Form } from './Form';
// ...50 more exports

// Consumer
import { Button } from './components'; // uses barrel

// Problem: bundlers sometimes evaluate the entire barrel to resolve the import,
// pulling in all 50+ components even though only Button is used.
// Depends on bundler version and whether the barrel is side-effect free.
```

```typescript
// ✅ Option A: Import directly from source (guaranteed tree-shaking)
import { Button } from './components/Button';

// ✅ Option B: Annotate barrel with sideEffects: false in package.json
// and use a bundler that handles ESM barrels well (Vite/Rollup handles this better than Webpack 4)

// ✅ Option C: Barrel with explicit re-exports + sideEffects: false declared
```

### 2. Class Methods

```typescript
// ❌ Class methods are NOT tree-shaken at the method level
class MathUtils {
  add(a: number, b: number) { return a + b; }
  subtract(a: number, b: number) { return a - b; }
  multiply(a: number, b: number) { return a * b; }
  divide(a: number, b: number) { return a / b; }
}

export const math = new MathUtils();

// Even if you only use math.add(), all methods ship — classes are atomic.

// ✅ Use standalone functions for tree-shakeable utilities
export function add(a: number, b: number) { return a + b; }
export function subtract(a: number, b: number) { return a - b; }
// Only imported functions are bundled
```

### 3. Object Exports

```typescript
// ❌ Object export — bundler includes all properties
export const utils = {
  formatDate: (d: Date) => d.toISOString(),
  parseDate: (s: string) => new Date(s),
  // ...
};

// Consumer uses: utils.formatDate — but parseDate ships too

// ✅ Named exports — fully tree-shakeable
export function formatDate(d: Date) { return d.toISOString(); }
export function parseDate(s: string) { return new Date(s); }
```

### 4. Dynamic Imports Used as Synchronous Requires

```typescript
// ❌ Pattern that defeats tree shaking
const { handler } = require(`./handlers/${type}`);
// Dynamic require — bundler can't know which handler will be needed

// This is a genuine CommonJS problem. In ESM you'd need:
const handler = await import(`./handlers/${type}`);
// Dynamic ESM import — still not tree-shakeable at build time
// (handled via code splitting instead — lazy chunk per handler)
```

### 5. Side-Effect Imports

```typescript
// ❌ This import is always included because it LOOKS like it has side effects
import './register-service-worker'; // even if nothing is used from it
// → file included in bundle because of implicit side effect

// ✅ If it truly has side effects (needs to run), that's correct
// ✅ If it exports something, import it properly and let tree-shaking work
```

## Verifying Tree Shaking Works

### Bundle Analysis

```bash
# Webpack: generate stats.json
npx webpack --profile --json > stats.json
# Then open https://webpack.github.io/analyse/ or use webpack-bundle-analyzer

# Vite: built-in rollup-plugin-visualizer
npm i -D rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';
plugins: [visualizer({ open: true, gzipSize: true })],
```

### Check for Unused Harmony Exports in Webpack Output

```bash
# Build with usedExports: true, minimize: false
# Grep for unused exports
grep -n "unused harmony export" dist/main.js
# Each line = a function/value that survived despite being marked unused
# (Terser removes these — but checking at this stage reveals what survived marking)
```

### Practical Test

```typescript
// Before: import from lodash (CommonJS — not tree-shakeable)
import { debounce } from 'lodash';
// Bundle includes entire lodash (~70KB gzipped)

// After: import from lodash-es (ESM — tree-shakeable)
import { debounce } from 'lodash-es';
// Bundle includes only debounce + its dependencies (~2KB gzipped)

// Measure with: npx source-map-explorer dist/main.js
```

## Real-World Package Tree-Shaking Guide

```
Package          Tree-Shakeable?   Notes
───────────────────────────────────────────────────────────────────
lodash           ❌ No            CommonJS — use lodash-es
lodash-es        ✅ Yes           ESM version of lodash
date-fns         ✅ Yes           ESM, sideEffects: false
moment           ❌ No            CommonJS, large locale data
dayjs            ✅ Yes (mostly)  ESM core, plugins need care
rxjs             ✅ Yes           Designed for tree-shaking
@mui/material    ⚠️  Partial     Named imports work; default imports don't
antd             ✅ v5+          ESM, requires babel-plugin-import for v4
@heroicons/react ✅ Yes           Individual icon imports
lucide-react     ✅ Yes           Named imports tree-shake well
```

## Common Mistakes

### 1. Forgetting sideEffects: false in Your Own Package

```json
// ❌ Publishing a utility library without sideEffects declaration
{
  "name": "my-utils",
  "main": "./dist/index.cjs.js",
  "module": "./dist/index.esm.js"
  // No sideEffects field → consumers can't tree-shake your library!
}

// ✅ Always declare sideEffects for utility libraries
{
  "name": "my-utils",
  "sideEffects": false,
  "main": "./dist/index.cjs.js",
  "module": "./dist/index.esm.js"
}
```

### 2. Using require() in ESM Packages

```typescript
// ❌ Mixing CJS require into an ESM package kills tree shaking
const lodash = require('lodash'); // defeats the purpose
export function debounce(fn: Function, wait: number) {
  return lodash.debounce(fn, wait);
}

// ✅ Stay pure ESM
import { debounce as _debounce } from 'lodash-es';
export function debounce(fn: Function, wait: number) {
  return _debounce(fn, wait);
}
```

### 3. Exporting Everything from index.ts

```typescript
// ❌ Re-exporting non-tree-shakeable modules through your barrel
// index.ts
export * from './utils';       // fine if utils is ESM
export * from './legacy';      // legacy.js uses require() → poisons entire barrel
```

## Interview Questions

### 1. Why does tree shaking require ES Modules?

**Answer:** Tree shaking requires static analysis of the import graph at build time. ES Modules use `import` declarations which must be at the top level and use string literals — the bundler can read them before executing any code, build a complete dependency graph, and determine which exports are referenced. CommonJS `require()` is a runtime function call — it can appear anywhere, use variables, be conditional. There's no way to know which exports are used without executing the code. Bundlers treat CJS modules as opaque blocks that must be included in full.

### 2. What is the sideEffects field in package.json and why does it matter?

**Answer:** `sideEffects` tells bundlers whether importing a module causes observable changes beyond its exported API (polyfilling globals, registering event listeners, writing to `window`). With `"sideEffects": false`, a bundler knows it's safe to skip a module entirely if none of its exports are used. Without this declaration, the bundler conservatively includes every file that's imported anywhere in the dependency chain, even if all exports are unused. This is the single most common reason a library looks tree-shakeable (ESM, named exports) but its code still ships in full — the package.json declaration is missing.

### 3. Why can't class methods be tree-shaken?

**Answer:** Classes are treated as a single unit — a class definition must be included or excluded entirely. A bundler cannot emit "the class but without these methods" because methods can be accessed dynamically (`instance['methodName']`), overridden in subclasses, or used by internal methods that are used. Standalone exported functions are the atomic unit of tree shaking. Libraries designed for tree shaking (like `date-fns`, `lodash-es`) expose utilities as individual named function exports rather than class instances for this reason.

### 4. How do barrel files (index.ts) affect tree shaking?

**Answer:** Barrel files that re-export everything from many modules can defeat tree shaking if the bundler can't prove each re-exported module is side-effect free — it may evaluate the entire barrel to resolve a single import. Webpack 4 was particularly bad at this; Rollup/Vite handle it better. Solutions: (1) import directly from source files rather than barrels, (2) ensure the package declares `"sideEffects": false`, (3) use a more modern bundler that handles ESM barrels well. The tradeoff is developer experience (barrels are convenient) vs tree-shaking reliability.

### 5. How would you verify that tree shaking is actually working for a given import?

**Answer:** Build with `minimize: false` (keep dead code readable), and search for `/* unused harmony export */` comments in the output — these are exports Webpack marked unused but hasn't removed yet (Terser removes them in production). Use `webpack-bundle-analyzer` or `rollup-plugin-visualizer` to visualize which modules contribute to the bundle and how much. For a specific library, compare bundle size with a known import: import one function from lodash-es vs importing from lodash — the ESM version should be dramatically smaller. `source-map-explorer dist/main.js` shows a treemap of what's actually in the bundle.
