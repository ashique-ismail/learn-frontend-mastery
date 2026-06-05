# Tree Shaking Prerequisites

## The Idea

**In plain English:** Tree shaking is how a build tool automatically removes unused code from your app before it ships to users. Think of it like a library that only packs the chapters you actually read into your bag — everything else stays on the shelf.

**Real-world analogy:** Imagine ordering a meal-prep kit that ships only the exact ingredients for the recipes you chose, instead of the entire warehouse inventory.

- The warehouse full of ingredients = the full JavaScript library with all its exported functions
- The recipes you chose = the specific imports your code uses
- The packing algorithm that reads your order = the bundler analyzing which exports are actually needed
- The final box shipped to your door = the trimmed bundle sent to the browser

---

## Overview

Tree shaking is a form of dead code elimination that removes unused exports from a JavaScript bundle. The term, popularized by Rollup, describes the process of statically analyzing the import/export graph, identifying which exported bindings are never imported, and omitting them from the output. The result is a smaller bundle containing only the code your application actually uses.

However, tree shaking is not automatic or guaranteed. It depends on a specific set of prerequisites — in the module format, the code itself, and the bundler configuration. Violating any of them silently disables tree shaking and can result in shipping hundreds of kilobytes of dead code to users. Understanding exactly why tree shaking succeeds or fails is a senior-level concern.

## Why Tree Shaking Requires Static Analysis

Tree shaking is fundamentally a static analysis problem. For a bundler to eliminate an export, it must prove at build time — without executing the code — that no reachable code path ever imports that export.

CommonJS `require()` is a runtime function call:

```javascript
// Bundler cannot know at build time what this imports
const feature = process.env.FEATURE;
const mod = require(`./${feature}`);
mod.doSomething();
```

ESM `import` statements are static declarations:

```javascript
// Bundler knows exactly what is imported, unconditionally
import { format, parseISO } from 'date-fns';
// If parseISO is never called, the bundler can remove it
```

This is why **ESM is a prerequisite for tree shaking**. CJS modules cannot be tree-shaken in the traditional sense.

## Prerequisite 1: ESM Syntax Throughout the Chain

### The Full Chain Must Use ESM

Tree shaking fails silently if any module in the dependency chain uses CJS. The entire subgraph being shaken must consist of ESM.

```javascript
// utils/index.js — uses ESM exports
export function used() { return 'used'; }
export function unused() { return 'unused'; } // should be eliminated
```

```javascript
// main.js
import { used } from './utils/index.js';
used();
// 'unused' is not imported — bundler can eliminate it
```

```javascript
// What happens if utils/index.js were CJS:
module.exports = {
  used: () => 'used',
  unused: () => 'unused'
};

// Now main.js:
const { used } = require('./utils/index.js');
// Bundler must include ALL of module.exports — cannot eliminate 'unused'
```

### Library package.json Must Expose ESM

When consuming third-party packages, check whether they ship ESM. Many packages still ship only CJS. The `exports` field in `package.json` indicates ESM availability:

```json
{
  "name": "my-library",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  },
  "module": "./dist/index.mjs"
}
```

The `"module"` field (a Rollup/webpack convention, not a Node.js standard) and `"exports"."import"` both point bundlers to the ESM build. Without one of these, bundlers fall back to the CJS `"main"` field and lose tree-shaking capability.

### Checking Whether a Package Is Tree-Shakeable

```bash
# Check if a package ships ESM
cat node_modules/lodash-es/package.json | grep '"module"'
cat node_modules/lodash/package.json    # No "module" field — CJS only, not tree-shakeable

# bundle-phobia CLI
npx bundle-phobia lodash    # shows tree-shakeable: false
npx bundle-phobia lodash-es # shows tree-shakeable: true
```

## Prerequisite 2: Named Exports (Not Default Object Exports)

### Default Object Exports Break Tree Shaking

```javascript
// utils.js — BAD for tree shaking
export default {
  format: (date) => { /* ... */ },
  parseISO: (str) => { /* ... */ },
  differenceInDays: (a, b) => { /* ... */ }
};
```

```javascript
// consumer.js
import utils from './utils.js';
utils.format(new Date()); // Only uses 'format'

// Bundler sees one default import — must include entire object
// parseISO and differenceInDays cannot be eliminated
```

```javascript
// utils.js — GOOD for tree shaking
export function format(date) { /* ... */ }
export function parseISO(str) { /* ... */ }
export function differenceInDays(a, b) { /* ... */ }
```

```javascript
// consumer.js
import { format } from './utils.js';
format(new Date());

// Bundler sees only 'format' imported — eliminates parseISO, differenceInDays
```

### Re-exports Must Also Use Named Exports

```javascript
// index.js — barrel file, BAD
export { default as format } from './format.js';
export { default as parseISO } from './parseISO.js';

// This can work if bundlers handle it, but causes issues (see barrel file section)
```

```javascript
// index.js — barrel file, GOOD
export { format } from './format.js';
export { parseISO } from './parseISO.js';
export { differenceInDays } from './differenceInDays.js';
```

## Prerequisite 3: Side-Effect-Free Code

### What is a Side Effect?

A side effect is any code that modifies state outside its own scope when the module is evaluated. Bundlers must retain modules with side effects even if their exports are unused, because those effects may be necessary for correct program behavior.

```javascript
// Side effects at module scope — bundler CANNOT remove this module
window.globalRegistry = {};                 // Assigns to global
document.addEventListener('DOMContentLoaded', init); // Registers handler
Array.prototype.flatMap = function() { /* polyfill */ }; // Mutates prototype
console.log('module loaded');               // I/O side effect
fetch('/api/init');                         // Network request
```

```javascript
// No side effects at module scope — safely tree-shakeable
export function add(a, b) { return a + b; }
export const PI = 3.14159;
// Pure exports only — if nothing imports them, they're eliminated
```

### The sideEffects Field in package.json

The `"sideEffects"` field in `package.json` is a direct optimization hint to bundlers:

```json
{
  "name": "my-library",
  "sideEffects": false
}
```

Setting `"sideEffects": false` tells bundlers that no module in this package has side effects at the module level. The bundler can safely remove any module whose exports are unused.

```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js",
    "./src/globalStyles.js"
  ]
}
```

Use an array to whitelist only the files that do have side effects. This allows tree shaking everywhere else.

```javascript
// Without sideEffects: false, this import cannot be tree-shaken
import './side-effects-unknown.js'; // bundler keeps it just in case
```

### Pure Annotations

For individual expressions, you can mark them pure for bundlers that support it:

```javascript
// Mark IIFE or function call as pure (no side effects)
const result = /*#__PURE__*/ expensiveComputation();

// Rollup and Webpack respect this comment
// If 'result' is never used, the entire call is eliminated
```

This is commonly seen in compiled TypeScript or Babel output for class helpers:

```javascript
// Compiled class with pure annotation
const MyClass = /*#__PURE__*/ (function() {
  function MyClass() {}
  MyClass.prototype.method = function() {};
  return MyClass;
})();
```

## Prerequisite 4: No Barrel File Anti-Patterns

### Barrel Files and Tree Shaking

Barrel files (index files that re-export from many submodules) are a common pattern that can impede tree shaking depending on how they're written and how the bundler handles them.

```javascript
// src/components/index.js — barrel file
export { Button } from './Button';
export { Modal } from './Modal';
export { Tooltip } from './Tooltip';
export { DataTable } from './DataTable'; // 50KB of dependencies
// ... 30 more components
```

```javascript
// app.js
import { Button } from './components'; // Only needs Button
// Older or misconfigured bundlers may include all 30 components
```

Modern bundlers (Vite, Webpack 5+, Rollup) handle well-written barrel files correctly — they follow the re-export chain and only include `Button`. However, if any component in the barrel has module-level side effects, the bundler must load that entire file.

### When Barrel Files Cause Issues

```javascript
// Problematic barrel — some exports have side effects
export { Button } from './Button'; // pure
export { initAnalytics } from './Analytics'; // Analytics.js has side effects!
// Now importing Button forces Analytics.js to load too
```

```javascript
// Also problematic — dynamic re-export
export * from './utils'; // Bundler may not analyze sub-exports of utils
```

### Recommendation: Import Directly When Size Matters

```javascript
// Guaranteed tree shaking — bypasses barrel
import { Button } from './components/Button';
import { Modal } from './components/Modal';
```

## Prerequisite 5: Bundler Configuration

### Rollup (Best Tree-Shaking Support)

Rollup was designed around ESM and has the best tree-shaking by default:

```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  treeshake: {
    // Preset: 'recommended' or 'safest'
    preset: 'recommended',
    
    // Mark all modules as side-effect-free (use with caution)
    moduleSideEffects: false,
    
    // Treat property access as side-effect-free
    propertyReadSideEffects: false,
    
    // Consider annotations
    annotations: true
  }
};
```

### Webpack 5

Webpack 5 has solid tree shaking with `mode: 'production'`:

```javascript
// webpack.config.js
module.exports = {
  mode: 'production', // enables tree shaking automatically
  optimization: {
    usedExports: true,    // marks unused exports
    minimize: true,       // minifier removes unused code
    sideEffects: true,    // reads package.json sideEffects field
    
    // Inner graph analysis — tracks which calls use which imports
    innerGraph: true
  }
};
```

Without `mode: 'production'`, tree shaking is disabled. Setting `optimization.usedExports: true` alone marks exports as unused but requires a minifier to actually remove the code.

### Vite

Vite uses Rollup for production builds, so tree shaking is Rollup-quality:

```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      // Rollup treeshake options apply
      treeshake: {
        preset: 'recommended'
      }
    }
  }
};
```

## Verifying Tree Shaking Works

### Check Bundle Output

```bash
# Build with Rollup and inspect
npx rollup src/index.js --file dist/bundle.js --format esm

# Use bundle analyzer for Webpack
npm install --save-dev webpack-bundle-analyzer
# Add to webpack config and run build
```

### Manual Test Pattern

```javascript
// test-tree-shake.js
import { onlyThis } from './your-library';
onlyThis();

// Build this file and check if other exports appear in the output
```

```bash
# Rollup analysis
npx rollup test-tree-shake.js --format esm | grep 'neverImported'
# Should produce no output if tree shaking works
```

### ESM Check Tools

```bash
# Check if a package is ESM and tree-shakeable
npx are-the-types-wrong my-package
npx publint my-package
```

## Comparison Table

| Prerequisite | Why It's Required | How to Verify |
| --- | --- | --- |
| ESM syntax | Static analysis requires static import/export | Check `"module"` or `"exports"."import"` in package.json |
| Named exports | Default object exports bundle entire object | Audit export patterns in source |
| sideEffects: false | Allows bundler to skip unused files | Check package.json, add if missing |
| No module-level side effects | Bundler must retain modules with side effects | Review module top-level code |
| Bundler in production mode | Tree shaking disabled in dev mode | Verify `mode: 'production'` or equivalent |
| Minification enabled | Marks removal + actually deletes dead code | Confirm terser/esbuild is in use |

## Best Practices

### 1. Always Set sideEffects in Your Library's package.json

```json
{
  "sideEffects": ["*.css"]
}
```

Even if you believe your code is pure, explicitly declaring it lets consumers' bundlers optimize aggressively.

### 2. Prefer lodash-es Over lodash

```javascript
// lodash — CJS, entire library bundled: ~70KB gzipped
import { cloneDeep } from 'lodash';

// lodash-es — ESM, only cloneDeep and its dependencies: ~5KB gzipped
import { cloneDeep } from 'lodash-es';
```

### 3. Audit Imports With Bundle Analysis

Run a bundle analyzer after every major dependency addition. A single poorly-tree-shaken import can add hundreds of kilobytes.

### 4. Avoid Class Inheritance With Side Effects

TypeScript class decorators and some mixin patterns generate code with side effects. Use `/*#__PURE__*/` annotations where safe.

### 5. Prefer Pure Functions Over Classes for Utilities

```javascript
// Classes with static methods are harder to tree-shake
class StringUtils {
  static capitalize(s) { return s[0].toUpperCase() + s.slice(1); }
  static trim(s) { return s.trim(); }
}

// Pure exported functions are straightforward to tree-shake
export function capitalize(s) { return s[0].toUpperCase() + s.slice(1); }
export function trim(s) { return s.trim(); }
```

## Interview Questions

### Q1: Why can't CommonJS modules be tree-shaken?

**Answer:** Tree shaking requires the bundler to statically determine which exports are used before executing any code. CommonJS `require()` is a runtime function call — it can accept computed strings, appear inside conditionals and loops, and be called with arguments that are only known at runtime. A bundler cannot know at build time which properties of `module.exports` will be accessed or what the full set of consumed exports is. ESM `import` statements are static declarations constrained to the top level, making the complete dependency and export usage graph knowable at parse time.

### Q2: What is the sideEffects field in package.json and how does it affect bundling?

**Answer:** `"sideEffects"` is a package.json field that tells webpack and other bundlers whether any module in the package executes code with observable side effects (I/O, global mutations, etc.) when evaluated. Setting it to `false` means no module has side effects; setting it to an array whitelists specific files that do. Without this hint, bundlers must conservatively assume every module might have side effects and cannot eliminate modules whose exports go unused. With `"sideEffects": false`, a bundler can skip loading any module whose exports are never consumed, even if that module is re-exported through a barrel file.

### Q3: You import only `Button` from a large component library's index.js barrel file but your bundle includes all 50 components. What are the possible causes?

**Answer:** Several things can prevent tree shaking here. First, if the library ships only CJS (`"main"` but no `"module"` or `"exports"."import"`), the bundler includes all of `module.exports`. Second, if any component in the barrel has module-level side effects (global registrations, CSS-in-JS injection, etc.), the bundler must retain the entire file. Third, if `"sideEffects"` is not set in the library's `package.json`, the bundler conservatively keeps all re-exported modules. Fourth, if the barrel uses `export * from` patterns, some bundlers may not trace the transitive graph. Solutions: import directly from the sub-path (`./components/Button`), check if an ESM build exists, or file an issue with the library maintainer about `"sideEffects": false`.

### Q4: What does the `/*#__PURE__*/` annotation do?

**Answer:** It is a hint to bundlers (supported by Rollup, Webpack's terser, and esbuild) that the annotated function call or expression has no side effects and its result is safe to eliminate if unused. When a bundler or minifier sees `/*#__PURE__*/` before a function call, it treats that call as side-effect-free and will remove it — along with the entire expression — if the result is never referenced. This is particularly useful for compiled TypeScript class helpers and higher-order function wrappers that look like function calls but are structurally pure.

### Q5: Does tree shaking work inside a single file, or only across module boundaries?

**Answer:** It works at both levels, but in different ways. Across module boundaries, tree shaking removes entire modules or named exports that are never imported. Within a single file (local dead code), most bundlers rely on the minifier (Terser, esbuild) to remove unused variables, unreachable branches, and code guarded by constants. Rollup performs some intra-file optimization during its scope-hoisting phase ("scope hoisting" flattens modules into a single scope and then eliminates unneeded declarations). The most aggressive dead code elimination within a file typically comes from minifiers, not the bundler's tree-shaking phase.

## Common Pitfalls

### 1. Assuming Production Mode Is Active

```bash
# Webpack in development mode has tree shaking disabled
# Always verify your build is using mode: 'production'
webpack --mode production

# Or check NODE_ENV
process.env.NODE_ENV === 'production' // must be true
```

### 2. Importing from CJS-Only Packages and Expecting Shaking

```javascript
// lodash is CJS — entire 70KB included regardless
import { pick } from 'lodash';

// lodash-es is ESM — only pick (~1KB) included
import { pick } from 'lodash-es';
```

### 3. Re-exporting Everything With export *

```javascript
// Barrel that uses export * — harder for bundlers to analyze
export * from './components';
export * from './utils';
export * from './hooks';

// Explicit named re-exports are safer
export { Button, Input } from './components';
```

### 4. Marking CSS Imports as Side-Effect-Free

```json
{
  "sideEffects": false
}
```

If your package imports CSS files and you declare `"sideEffects": false`, CSS imports will be eliminated. Always whitelist CSS:

```json
{
  "sideEffects": ["**/*.css", "**/*.scss"]
}
```

## Resources

### Official Documentation

- [Webpack: Tree Shaking](https://webpack.js.org/guides/tree-shaking/)
- [Rollup: Tree-shaking](https://rollupjs.org/faqs/#why-is-tree-shaking-not-working)
- [Vite: Build Optimizations](https://vitejs.dev/guide/features#build-optimizations)

### Articles and Guides

- [How Rollup Tree Shaking Works](https://medium.com/@Rich_Harris/tree-shaking-versus-dead-code-elimination-d3765df85c80)
- [Webpack sideEffects Docs](https://webpack.js.org/configuration/optimization/#optimizationsideeffects)
- [Pure ESM package guide](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c)

### Tools

- [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
- [rollup-plugin-visualizer](https://github.com/btd/rollup-plugin-visualizer)
- [bundlephobia.com](https://bundlephobia.com) — checks if packages are tree-shakeable
- [publint](https://publint.dev) — lints package.json for correct ESM/CJS setup
