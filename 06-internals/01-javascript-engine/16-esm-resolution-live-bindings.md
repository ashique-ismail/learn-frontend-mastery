# ESM Module Resolution, Live Bindings, and Circular Dependencies

## The Idea

**In plain English:** JavaScript files can share code with each other by "exporting" things from one file and "importing" them in another — ESM (ECMAScript Modules) is the official system for doing this, where the imported values stay permanently connected to the original source so any update is instantly visible everywhere.

**Real-world analogy:** Imagine a scoreboard at a sports stadium that is wired live to the official scorekeeping booth. Every fan in the stadium sees the current score at all times — not a printed handout that was correct when it left the printer.

- The scoreboard display = the imported variable in a consuming module
- The official score in the booth = the exported variable in the source module
- The live wire connecting them = the live binding
- A printed handout given out before the game = a CommonJS `require()` value copy (frozen at the moment it was taken)

---

## Table of Contents
- [Overview](#overview)
- [ESM vs CommonJS Fundamentals](#esm-vs-commonjs-fundamentals)
- [The ESM Resolution Algorithm](#the-esm-resolution-algorithm)
- [Module Loading Phases](#module-loading-phases)
- [Live Bindings](#live-bindings)
- [Circular Dependency Handling](#circular-dependency-handling)
- [Import Maps and Browser Resolution](#import-maps-and-browser-resolution)
- [Dynamic Import](#dynamic-import)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

ECMAScript Modules (ESM) are a fundamentally different execution model from CommonJS (CJS). The distinction is not merely syntactic (`import` vs `require`). ESM modules are **statically analyzable**, **lazily evaluated**, and export **live bindings** — not copied values. Understanding these properties explains why tree-shaking works, why circular dependencies behave differently in ESM vs CJS, and why certain patterns that work in CJS silently break in ESM.

```
ESM properties:
  ✓ Statically analyzable imports (no dynamic paths in static imports)
  ✓ Live bindings (exports reflect current value, not snapshot)
  ✓ Deferred evaluation (module code runs once, lazily on first import)
  ✓ Asynchronous loading (host may fetch modules concurrently)
  ✓ Strict mode always (no need for 'use strict')
  ✓ Top-level await supported (ES2022)
  ✗ No __dirname, __filename (use import.meta.url instead)
  ✗ No require() (use dynamic import() instead)
```

## ESM vs CommonJS Fundamentals

### Export Models

```typescript
// ── CommonJS ──────────────────────────────────────────────────────────────
// CJS exports a VALUE snapshot at the time require() is called.
// Mutations to the exported object are shared, but re-assignments are not.

// counter.js (CJS)
let count = 0;
function increment() { count++; }
module.exports = { count, increment };
// Exports a COPY of count (value 0 at export time)

// consumer.js (CJS)
const { count, increment } = require('./counter');
increment();
console.log(count); // STILL 0 — got a copy of the value, not a reference

// ── ESM ─────────────────────────────────────────────────────────────────
// ESM exports a LIVE BINDING — a reference to the export's memory location.

// counter.mjs (ESM)
export let count = 0;
export function increment() { count++; }

// consumer.mjs (ESM)
import { count, increment } from './counter.mjs';
increment();
console.log(count); // 1 — live binding reflects updated value
```

### Static vs Dynamic Shape

```typescript
// ESM: All imports must be at the top level, paths must be string literals
// (static form — enables static analysis and tree-shaking)
import { readFile } from 'node:fs/promises'; // ✓ static
import defaultExport from './module.js';      // ✓ static

// These are SYNTAX ERRORS in ESM static imports:
// if (condition) import './mod.js';          // ✗ conditional
// import(`./handlers/${name}.js`);           // ✗ dynamic path in static import

// CJS: require() is just a function — fully dynamic
const mod = require(condition ? './a' : './b'); // ✓ legal in CJS
const name = 'handler';
const handler = require(`./handlers/${name}`);  // ✓ legal in CJS
// But this means bundlers cannot statically analyze what is imported!
```

## The ESM Resolution Algorithm

### Node.js ESM URL Resolution

Node.js resolves ESM specifiers using the following algorithm (condensed from the Node.js ESM spec):

```
RESOLVE(specifier, parentURL):
  1. If specifier starts with "/" or "./" or "../":
     → relative URL resolution against parentURL
     → return file URL

  2. If specifier is a valid URL (e.g., "file:///...", "node:fs"):
     → return specifier as-is

  3. Otherwise (bare specifier, e.g., "lodash"):
     → look in package.json "exports" field of resolved package
     → fall back to node_modules resolution:
       a. Check parentURL's node_modules/specifier
       b. Walk up directory tree checking node_modules/ at each level
       c. If not found → ERR_MODULE_NOT_FOUND

  4. For file paths:
     → Do NOT add file extensions automatically (unlike CJS!)
     → If path has no extension → ERR_MODULE_NOT_FOUND
     → If path is a directory → check package.json "main"/"exports"
```

```typescript
// ── File Extension Required in ESM ─────────────────────────────────────────
// CJS: require('./utils') tries utils.js, utils/index.js, etc.
// ESM: import './utils' is an error — exact URL must be provided

// WRONG in Node ESM:
import { helper } from './utils'; // ERR_MODULE_NOT_FOUND

// CORRECT:
import { helper } from './utils.js'; // Must include .js extension
// (Even when using TypeScript, you write .js — the TS compiler resolves it)

// TypeScript ESM: write .js extension, TS resolves to .ts file at compile time
import { helper } from './utils.js'; // Resolves to utils.ts in TS project
```

### Package Exports Field

The `exports` field in `package.json` is the modern way to control which files are accessible from a package:

```json
// package.json
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs",
      "types": "./dist/types/index.d.ts"
    },
    "./utils": {
      "import": "./dist/esm/utils.js",
      "require": "./dist/cjs/utils.cjs"
    },
    "./package.json": "./package.json"
  }
}
```

```typescript
// Consumers of my-lib:
import { fn } from 'my-lib';           // Resolves to ./dist/esm/index.js
import { util } from 'my-lib/utils';  // Resolves to ./dist/esm/utils.js

// Deep imports blocked by exports field:
import something from 'my-lib/src/internal'; // ERR_PACKAGE_PATH_NOT_EXPORTED
// Before exports field existed, this would have worked — breaking encapsulation
```

### Import Conditions

```json
// Conditional exports support multiple environments
{
  "exports": {
    ".": {
      "development": "./dist/dev.js",       // Custom condition
      "production": "./dist/prod.js",       // Custom condition
      "browser": "./dist/browser.js",       // Browser-specific
      "node": "./dist/node.js",             // Node-specific
      "import": "./dist/esm/index.js",      // ESM consumers
      "require": "./dist/cjs/index.cjs",    // CJS consumers
      "default": "./dist/esm/index.js"      // Fallback
    }
  }
}
```

```bash
# Node.js custom conditions
node --conditions=development app.js
```

## Module Loading Phases

ESM loading is a three-phase process — unlike CJS which is synchronous and single-pass:

```
Phase 1: CONSTRUCTION (parse and link)
  ├── Fetch module source (network/disk)
  ├── Parse source → find all import declarations
  ├── Recursively construct all dependencies
  └── Create Module Record with uninitialized export bindings

Phase 2: INSTANTIATION (link bindings)
  ├── Allocate memory locations for all exports
  ├── Wire up import bindings to export memory locations
  └── Live binding connections established (exports are "live" references)

Phase 3: EVALUATION (execute)
  ├── Execute module code top-to-bottom (once, ever)
  ├── Export bindings get their values assigned
  └── Return module namespace object
```

```typescript
// This three-phase model means:
// 1. All import/export names are known BEFORE any code runs
// 2. Circular dependencies are handled by reserving binding slots
// 3. A module's code runs exactly once regardless of import count

// Proof of single evaluation:
// a.mjs
console.log('a evaluated');
export const value = 42;

// b.mjs
import './a.mjs';
import './a.mjs'; // Same module — NOT evaluated twice
// Console only prints 'a evaluated' once

// c.mjs
import './a.mjs'; // Still NOT evaluated again
import './b.mjs'; // b imports a, but a is already in the module map
```

### Module Map (Module Registry)

```typescript
// The runtime maintains a Module Map keyed by URL:
// Map<URL string, ModuleRecord>
//
// First time URL is imported → fetch, parse, instantiate, evaluate
// Subsequent imports → return cached ModuleRecord immediately
//
// This is why:
// - Side effects run exactly once
// - Mutations to module state are shared across all importers
// - Circular dependencies don't cause infinite loops

// shared-state.mjs
export const registry = new Map(); // Created once, shared by all importers

// service-a.mjs
import { registry } from './shared-state.mjs';
registry.set('a', 'service-a');

// service-b.mjs
import { registry } from './shared-state.mjs';
registry.set('b', 'service-b');

// main.mjs
import { registry } from './shared-state.mjs';
import './service-a.mjs';
import './service-b.mjs';
console.log(registry); // Map { 'a' => 'service-a', 'b' => 'service-b' }
// All three imports got the SAME Map instance
```

## Live Bindings

### What "Live Binding" Means

A live binding is not a copy of a value — it is a **read-only reference to the export's memory slot** in the exporting module. When the exporting module updates the variable, all importing modules see the updated value through their live binding.

```
Memory model for live bindings:

Exporting module (counter.mjs):
  Memory slot [0x1a2b]: count = 0  ←────────────────┐
                                                      │ live reference
Importing module (consumer.mjs):                      │
  `count` identifier ────────────────────────────────┘
  (NOT a copy — reading `count` reads from [0x1a2b])

When counter.mjs does: count++ → [0x1a2b] becomes 1
When consumer.mjs reads: count → reads [0x1a2b] → gets 1
```

```typescript
// counter.mjs
export let count = 0;
export function increment(): void { count++; }
export function reset(): void { count = 0; }

// consumer.mjs
import { count, increment, reset } from './counter.mjs';

console.log(count); // 0
increment();
console.log(count); // 1 — live binding, sees update!
increment();
console.log(count); // 2
reset();
console.log(count); // 0 — live binding reflects reset

// The import IS READ-ONLY from the consumer's perspective:
count = 5; // TypeError: Assignment to constant variable
           // Consumer cannot write through the live binding
```

### Live Bindings Enable Late Binding Patterns

```typescript
// utils.mjs — utility that is updated after declaration
export let formatter = (n: number) => n.toFixed(2); // default

export function setFormatter(fn: (n: number) => string): void {
  formatter = fn; // Update the binding
}

// consumer.mjs
import { formatter, setFormatter } from './utils.mjs';

console.log(formatter(3.14159)); // "3.14"

// Reconfigure the formatter
setFormatter((n) => `$${n.toFixed(4)}`);

console.log(formatter(3.14159)); // "$3.1416" — consumer sees updated function!
// In CJS, this would NOT work — consumer has a copy of the old function ref
```

### CJS Compatibility Shim Gotcha

```typescript
// When importing CJS from ESM, Node.js creates a "default export" wrapper
// The entire module.exports object becomes the default export
// Named exports may or may not be available depending on static analysis

// utils.cjs (CommonJS)
let count = 0;
module.exports.count = count;
module.exports.increment = function() { count++; };

// consumer.mjs (ESM importing CJS)
import utils from './utils.cjs'; // default import = entire module.exports
import { count, increment } from './utils.cjs'; // Named imports (may work)

// CRITICAL: CJS exports are NOT live bindings!
// increment() updates count in utils.cjs but module.exports.count
// is still the original value (0) because it was set to the value, not a live ref
increment();
console.log(utils.count); // 0 — NOT a live binding in CJS interop!
console.log(count);       // 0 — same problem with named import
```

## Circular Dependency Handling

### CJS Circular Dependencies: Partial Exports

```javascript
// In CJS, circular deps return whatever has been exported SO FAR
// when the cycle is encountered (partial module).

// a.cjs
const { b } = require('./b.cjs');
console.log('a got b:', b); // may be undefined if b not yet exported!
exports.a = 'module-a';

// b.cjs
const { a } = require('./a.cjs');
console.log('b got a:', a); // undefined — a.cjs hasn't finished yet!
exports.b = 'module-b';

// require('./a.cjs') from main:
// 1. Start evaluating a.cjs
// 2. a.cjs requires b.cjs
// 3. Start evaluating b.cjs
// 4. b.cjs requires a.cjs → CYCLE DETECTED
// 5. Return partial exports of a.cjs (currently empty {})
// 6. b.cjs gets `a` = undefined
// 7. b.cjs finishes → exports.b = 'module-b'
// 8. a.cjs continues, gets `b` = 'module-b' (b finished)
// 9. a.cjs finishes
```

### ESM Circular Dependencies: Temporal Dead Zone

```typescript
// ESM uses live bindings to handle circular dependencies more gracefully
// BUT there's still a TDZ issue for uninitialized bindings

// a.mjs
import { b } from './b.mjs';
export const a = 'module-a';
console.log('a uses b:', b); // b is live — if b.mjs has run, this is fine

// b.mjs
import { a } from './a.mjs';
export const b = 'module-b';
console.log('b uses a:', a); // a is live but may be TDZ if a.mjs hasn't run!
```

```
ESM circular dependency evaluation order:

Construction phase (all modules):
  a.mjs: found, slot reserved for `a` (uninitialized)
  b.mjs: found, slot reserved for `b` (uninitialized)
  Live binding: b.mjs's `a` → a.mjs's `a` slot
  Live binding: a.mjs's `b` → b.mjs's `b` slot

Evaluation phase (depth-first, post-order):
  b.mjs evaluates first (it's deeper in the import graph from a.mjs)
  b.mjs tries to read `a` → slot is uninitialized → ReferenceError!

  UNLESS: a.mjs is higher in the import graph or `a` is a function
  (Functions are hoisted to their binding before evaluation starts)
```

### Safe Circular Dependency Pattern: Functions vs Values

```typescript
// a.mjs
import { getB } from './b.mjs';

export function getA(): string {
  return 'a-value';
}

// Note: getA is a function expression / declaration
// When b.mjs reads `getA` at evaluation time, it accesses the function
// which is defined. Functions work in circular deps; const/let values don't.

// b.mjs
import { getA } from './a.mjs';

export function getB(): string {
  // SAFE: getA is called at RUNTIME, not at module evaluation time
  // By the time this runs, a.mjs has fully evaluated
  return `b-using-${getA()}`;
}

// main.mjs
import { getA } from './a.mjs';
import { getB } from './b.mjs';

console.log(getB()); // 'b-using-a-value' — works because getA is called lazily
```

### Breaking Circular Dependencies with Lazy Initialization

```typescript
// PROBLEMATIC: Value-level circular dependency
// config.mjs
import { logger } from './logger.mjs';
export const config = { level: 'info', logger }; // reads logger at init time

// logger.mjs
import { config } from './config.mjs'; // reads config at init time
export const logger = { log: (m: string) => console.log(`[${config.level}] ${m}`) };
// Circular! One of these will see uninitialized binding.

// ── Solution 1: Lazy getter ─────────────────────────────────────────────────
// config.mjs
export const config = {
  level: 'info',
  get logger() { return import('./logger.mjs'); } // lazy, not at init time
};

// ── Solution 2: Third module breaks the cycle ───────────────────────────────
// types.mjs (no imports — pure types/constants)
export type LogLevel = 'debug' | 'info' | 'warn' | 'error';
export const DEFAULT_LEVEL: LogLevel = 'info';

// logger.mjs (imports from types.mjs only)
import { DEFAULT_LEVEL } from './types.mjs';
export const logger = { level: DEFAULT_LEVEL, log(m: string) { ... } };

// config.mjs (imports from both)
import { logger } from './logger.mjs';
export const config = { logger }; // No cycle!

// ── Solution 3: Defer the dependency ───────────────────────────────────────
// a.mjs — use factory function to defer creation
export function createA() {
  const { b } = require('./b.mjs'); // Only when createA() is called
  return { a: 'value', refToB: b };
}
// (Using dynamic import() in ESM for the same effect)
```

### Detecting Circular Dependencies

```bash
# Using madge (popular tool for visualizing module deps)
npx madge --circular src/index.ts
# Prints: Circular dependency found!
# a.ts → b.ts → c.ts → a.ts

# With circular visualization
npx madge --circular --image dep-graph.svg src/index.ts

# Node.js built-in: trace module loading
node --trace-warnings index.mjs
# Warns about circular dependency issues
```

## Import Maps and Browser Resolution

### Import Maps

Import maps (now baseline in all browsers) allow mapping bare specifiers to URLs without a bundler:

```html
<script type="importmap">
{
  "imports": {
    "lodash": "https://cdn.skypack.dev/lodash@4.17.21",
    "lodash/": "https://cdn.skypack.dev/lodash@4.17.21/",
    "react": "/node_modules/react/index.js",
    "#utils": "/src/utils.js"
  },
  "scopes": {
    "/legacy/": {
      "react": "https://unpkg.com/react@17/umd/react.development.js"
    }
  }
}
</script>

<script type="module">
  import _ from 'lodash';         // Maps to cdn URL
  import chunk from 'lodash/chunk'; // Maps to cdn URL + 'chunk'
  import { helper } from '#utils'; // Package self-reference
</script>
```

```typescript
// Browser ESM resolution without import map:
// import './utils' is resolved relative to document baseURL
// import 'lodash' throws — no node_modules in browser

// With import map: bare specifiers work in browser too
// This is how unbundled browser development becomes viable
```

### Module Worker and importScripts

```typescript
// Service Workers and Web Workers can use ESM
// Classic workers: importScripts() — synchronous, legacy
// Module workers: import — async, modern

// Classic (avoid):
// importScripts('./utils.js'); // Synchronous, blocks worker

// Modern (prefer):
const worker = new Worker('./worker.js', { type: 'module' });

// worker.js (module worker)
import { heavyCompute } from './compute.js'; // ESM import works here

self.onmessage = async (event) => {
  const result = await heavyCompute(event.data);
  self.postMessage(result);
};
```

## Dynamic Import

### import() Mechanics

`import()` is a syntactic form (not a function) that returns a `Promise<ModuleNamespace>`. It follows the same three-phase loading process as static imports but deferred to call time.

```typescript
// Static import — synchronous wiring, evaluated once
import { parseJSON } from './parser.js';

// Dynamic import — returns Promise, can be conditional/lazy
async function loadParser(type: 'json' | 'xml'): Promise<Parser> {
  const module = await import(`./parsers/${type}.js`);
  // ⚠️ Dynamic path: bundlers CANNOT statically analyze this
  // They'll include ALL parsers/*.js or none
  return module.default;
}

// ✓ Bundler-friendly dynamic import with static path
async function loadJSONParser(): Promise<JSONParser> {
  const { parseJSON } = await import('./parsers/json.js');
  return parseJSON;
}

// ✓ Lazy-loaded component pattern (React)
const LazyChart = React.lazy(() => import('./Chart.js'));
// Bundler sees the static string './Chart.js' and splits it to a chunk
```

### Top-Level Await

```typescript
// ESM supports top-level await (ES2022)
// The module is treated as if wrapped in an async function
// Importers await the module's completion

// database.mjs
const connection = await createConnection(process.env.DB_URL);
// ^ This suspends module evaluation until connection is established

export { connection };

// api.mjs
import { connection } from './database.mjs';
// api.mjs does NOT start evaluating until database.mjs's await resolves
// This is safe because ESM loading is async; CJS cannot do this

// Gotcha: Top-level await in a circular dep is a deadlock!
// a.mjs: await import('./b.mjs') — waits for b
// b.mjs: await import('./a.mjs') — waits for a → deadlock
```

### Import Assertions and Attributes (ES2024+)

```typescript
// Import JSON modules (now: import attributes with "with")
import data from './config.json' with { type: 'json' };

// Import CSS modules (experimental, stage 2)
import styles from './component.css' with { type: 'css' };

// Without assertion, importing non-JS raises an error in strict hosts
// The { type: 'json' } assertion tells the loader how to interpret the file
```

## Common Pitfalls

### Pitfall 1: Writing CJS-style code in ESM

```typescript
// BAD: CJS patterns in ESM files
// These do not exist in ESM module scope:
console.log(__dirname);  // ReferenceError: __dirname is not defined
console.log(__filename); // ReferenceError: __filename is not defined
const mod = require('./utils'); // ReferenceError: require is not defined

// GOOD: ESM equivalents
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Dynamic require → dynamic import
const mod = await import('./utils.js');
```

### Pitfall 2: Assuming ESM Re-exports Copy Values

```typescript
// TRAP: Re-exporting from ESM modules creates live binding chains
// counter.mjs
export let count = 0;
export const inc = () => count++;

// proxy.mjs — re-exporting
export { count, inc } from './counter.mjs';
// count is a live binding through proxy.mjs

// main.mjs
import { count, inc } from './proxy.mjs';
inc();
console.log(count); // 1 — live binding chains work end-to-end

// But export { count as default } creates a different behavior
// export default count  ← This takes a snapshot of count's VALUE at this point
// Not a live binding for the default export!
```

### Pitfall 3: Named vs Default Export Confusion with CJS Interop

```typescript
// lib.cjs (CommonJS)
module.exports = function myLib() { return 42; };
module.exports.helper = function() { return 'help'; };

// Importing CJS module in ESM:
import myLib from './lib.cjs';          // default = entire module.exports
import { helper } from './lib.cjs';     // Named export (static analysis may find it)

// The default export is the whole module.exports value:
console.log(myLib()); // 42 — works because module.exports is a function
console.log(myLib.helper()); // 'help' — property on the function

// This behavior differs from:
// const myLib = require('./lib.cjs'); // Same thing in CJS
```

### Pitfall 4: Tree-Shaking and Side Effects

```typescript
// package.json
{
  "sideEffects": false  // Tell bundlers: all files are side-effect free
  // OR
  "sideEffects": ["./src/polyfills.js", "*.css"] // Specific files have side effects
}

// BAD: Side-effectful code at module top level prevents tree-shaking
// analytics.mjs
window.analytics = createAnalytics(); // Side effect at top level!
export function track(event: string): void { window.analytics.track(event); }

// If a bundler tries to tree-shake and 'track' is not imported,
// it cannot safely eliminate analytics.mjs because it has a side effect

// GOOD: Move side effects into functions
// analytics.mjs
let analytics: Analytics | null = null;

export function initAnalytics(): void {
  analytics = createAnalytics(); // Only runs when explicitly called
}

export function track(event: string): void {
  analytics?.track(event);
}
// Now if initAnalytics() is not called, bundler can eliminate this module
```

## Interview Questions

### Question 1: What is a live binding and how does it differ from CJS exports?

**Answer**: In ESM, an import binding is a live read-only reference to the memory slot holding the exported variable in the exporting module. When the exporting module updates the variable, all importers see the updated value through their live binding — similar to passing a reference, not a value. In CommonJS, `module.exports` assigns values at execution time. When you destructure: `const { count } = require('./counter')`, you get a copy of `count`'s value at that moment. If the counter module later increments its internal `count`, your local copy stays unchanged. This is why `increment()` works across ESM live bindings but not CJS destructured exports.

### Question 2: Explain the three phases of ESM module loading.

**Answer**:
1. **Construction**: The loader recursively fetches and parses module source, building the full dependency graph. For each module, it creates a Module Record and identifies all imports/exports. No code runs yet. This phase is where errors like `SyntaxError` in imports are caught.
2. **Instantiation**: Memory slots are allocated for all exports. Import bindings are connected to their corresponding export slots — establishing the live binding connections. No code runs yet, but all the "wires" are connected.
3. **Evaluation**: Module code runs top-to-bottom, exactly once per module URL. Export bindings get their values. After evaluation, the module is cached — subsequent imports return the cached module record without re-evaluating.

### Question 3: How does ESM handle circular dependencies differently from CJS?

**Answer**: CJS handles circular deps by returning a partial `module.exports` object — whatever has been assigned at the time the cycle is detected. This means circular imports often get `undefined` values. ESM handles cycles through live bindings established in the instantiation phase. All module slots are reserved before any evaluation begins. If module A imports from module B which imports from A, the bindings are connected but may be in the Temporal Dead Zone (uninitialized) at evaluation time. ESM circular deps work correctly only if the consumed binding is accessed after the exporting module has evaluated — function calls at runtime (not at module top-level) are safe because the exporting module will have evaluated by then.

### Question 4: Why must file extensions be explicit in Node.js ESM imports?

**Answer**: Node.js ESM follows the URL specification rather than Node's traditional CJS path resolution. URLs are exact — there is no implicit extension resolution or directory index lookup. The CJS resolver's automatic `.js`, `/index.js`, `/index.json` probing is a Node-specific convention, not part of any URL standard. In ESM, `import './utils'` is a relative URL that must resolve to a file that exists at exactly that URL. Node intentionally removed the magic resolution to align with browser behavior, where `<script type="module" src="./utils">` would not try to append extensions. TypeScript works around this by allowing `.js` extensions in source that resolve to `.ts` files during compilation.

### Question 5: What does "statically analyzable" mean for ESM, and why does it matter for tree-shaking?

**Answer**: Static analyzability means that all `import` statements declare their dependencies with string literals at the top of the file, before any code runs. A bundler (or the V8 parser) can determine the full import/export graph without executing any code — no conditionals, no dynamic paths, no runtime evaluation needed. This enables **tree-shaking**: a bundler can build the complete dependency graph statically, mark every exported binding, trace which ones are actually used (reachability analysis), and eliminate all unused exports and their transitive dependencies. In CJS, `require()` can appear anywhere with any string value — including computed paths — making it impossible to statically determine what's imported. Bundlers can approximate CJS tree-shaking but cannot guarantee it.

## Key Takeaways

1. **Live bindings are references, not copies** — ESM imports reflect the current value of the exporting variable; destructured CJS exports are value snapshots

2. **Module evaluation happens exactly once** — the module map caches evaluated modules; multiple imports of the same specifier all get the same module instance

3. **ESM loading is three phases** — construction (parse), instantiation (link), evaluation (execute); all three complete before module code is considered "done"

4. **File extensions are required in Node.js ESM** — `'./utils'` is an error; `'./utils.js'` is correct; TypeScript resolves `.js` to `.ts` at compile time

5. **Circular dependencies work better with functions than values** — function calls are deferred to runtime (after all modules have evaluated); value reads at module top level may encounter TDZ uninitialized bindings

6. **Top-level await propagates** — a module that uses `await` at top level makes its importers async; combining TLA with circular deps risks deadlock

7. **Static imports enable tree-shaking** — bundlers can eliminate unused exports only when the import graph is statically knowable; dynamic paths disable this

8. **CJS interop uses a "default export" wrapper** — when importing CJS from ESM, the whole `module.exports` becomes the default export; named exports may be synthesized but are not live bindings

9. **The `exports` field in package.json is the modern encapsulation boundary** — it controls which subpaths are publicly importable and allows dual CJS/ESM packages

10. **Import maps bring Node-style resolution to the browser** — they allow bare specifiers in browser ESM without a bundler step

## Resources

### Specifications
- [ECMAScript Modules — TC39 Spec](https://tc39.es/ecma262/#sec-modules)
- [Node.js ESM Documentation](https://nodejs.org/api/esm.html)
- [WHATWG HTML — Module Loading](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier)
- [Import Maps — WICG](https://wicg.github.io/import-maps/)

### Articles
- "JavaScript Modules" — MDN Web Docs
- "ES modules: A cartoon deep-dive" — Lin Clark (Mozilla Hacks)
- "Node.js ESM Loaders" — Node.js Documentation

### Tools
- `madge` — visualize circular dependencies
- `rollup` — ESM-first bundler, excellent for tree-shaking analysis
- `es-module-lexer` — fast ESM import/export parser used by Vite
