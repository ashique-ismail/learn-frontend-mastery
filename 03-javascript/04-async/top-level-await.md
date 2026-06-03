# Top-Level Await

## Overview

Top-level await (TLA) allows the `await` operator to be used outside of an `async` function — directly at the top level of an ES module. Introduced in ES2022 and standardized as part of the ECMAScript specification, TLA changes how modules initialize, enabling dynamic import decisions, conditional polyfill loading, and resource acquisition as part of module evaluation.

Before TLA, async initialization in modules required wrapping everything in an async IIFE, which was awkward and obscured intent. TLA makes the module system itself understand async initialization.

## The Problem TLA Solves

### Pre-TLA Workarounds

Before top-level await, async setup code at module level required an IIFE wrapper:

```javascript
// Old pattern — cumbersome and misleading
let db;

(async () => {
  db = await connectToDatabase();
})();

// BUG: db is undefined here because the IIFE hasn't resolved yet
export function query(sql) {
  return db.execute(sql); // db may be undefined!
}
```

Or an exported initialization function:

```javascript
// Old pattern — consumers must remember to call init()
let db;

export async function init() {
  db = await connectToDatabase();
}

export function query(sql) {
  return db.execute(sql); // Still unsafe if init() wasn't called
}
```

Both patterns are error-prone. Top-level await makes initialization a first-class concern of the module system.

## Top-Level Await Syntax

In an ES module, `await` can be used directly at the module's top scope:

```javascript
// config.mjs
const response = await fetch('/api/config');
export const config = await response.json();

// db.mjs
export const db = await createConnection(config.databaseUrl);

// main.mjs
import { config } from './config.mjs';   // Awaits config.mjs resolution
import { db } from './db.mjs';           // Awaits db.mjs resolution

// Both config and db are fully initialized here
console.log(config.appName);
db.query('SELECT 1');
```

## How Module Loading Is Affected

### Module Graph Evaluation

TLA does not block the entire JavaScript thread. When a module uses top-level await, it becomes an "asynchronous module" in the module graph:

- Sibling modules with no dependency on the awaiting module continue to execute
- Modules that import from the awaiting module wait for it to complete
- The parent module that imports the async module also becomes implicitly async

```javascript
// a.mjs — awaits for 1 second
const value = await delay(1000);
export { value };

// b.mjs — no TLA, no dependency on a.mjs
export const greeting = 'hello'; // Evaluates immediately

// main.mjs — imports both
import { value } from './a.mjs';   // Must wait ~1s for a.mjs
import { greeting } from './b.mjs'; // Available immediately

// main.mjs itself is implicitly async — the host (browser/Node) awaits it
```

The browser's module loading algorithm (specified in HTML) coordinates this: sibling imports are evaluated in parallel where possible, but a module's body doesn't run until all its dependencies are resolved.

### Comparison with Synchronous Modules

```javascript
// sync-module.mjs — evaluated synchronously
export const X = computeValue(); // Runs immediately at evaluation time

// async-module.mjs — evaluation may be deferred
const data = await fetchData(); // Module evaluation pauses here
export const X = process(data); // Runs after fetch completes
```

Importing `async-module.mjs` changes the importing module to also be asynchronous (it must wait), propagating up the import graph.

## Use Cases

### Dynamic Feature Detection

```javascript
// polyfills.mjs
if (!globalThis.structuredClone) {
  const { structuredClone } = await import('./structuredClone-polyfill.mjs');
  globalThis.structuredClone = structuredClone;
}
```

### Conditional Locale/Resource Loading

```javascript
// i18n.mjs
const locale = navigator.language.split('-')[0];

const messages = await import(`./locales/${locale}.mjs`)
  .catch(() => import('./locales/en.mjs')); // Fallback to English

export const t = (key) => messages[key] ?? key;
```

### Database and Service Initialization

```javascript
// services/database.mjs
import { createPool } from 'mysql2/promise';
import { config } from '../config.mjs';

const pool = await createPool({
  host: config.db.host,
  user: config.db.user,
  password: config.db.password,
  database: config.db.name,
  connectionLimit: config.db.poolSize,
});

// Verify connection on startup
await pool.query('SELECT 1');

export { pool };
```

Any module that imports `database.mjs` is guaranteed to have a working, tested connection.

### Environment-Specific Configuration

```javascript
// config.mjs
const env = process.env.NODE_ENV ?? 'development';

const { default: envConfig } = await import(`./config.${env}.mjs`);
const { default: baseConfig } = await import('./config.base.mjs');

export const config = { ...baseConfig, ...envConfig };
```

### Lazy Initialization of Heavy Dependencies

```javascript
// image-processor.mjs
// Only load the heavy WebAssembly module when this module is imported
const { default: wasm } = await import('./image-processing.wasm');
await wasm.initialize();

export function processImage(buffer) {
  return wasm.process(buffer);
}
```

## Restrictions and Requirements

### ES Modules Only

Top-level await is only valid in ES modules (`.mjs` files, or `.js` with `"type": "module"` in `package.json`, or scripts with `type="module"`). Using it in CommonJS (`require`) contexts throws a `SyntaxError`.

```javascript
// CommonJS — does NOT support TLA
const data = await fetch('/api'); // SyntaxError: await is only valid in async functions
```

```html
<!-- Only works with type="module" -->
<script type="module">
  const data = await fetch('/api/config').then(r => r.json());
  init(data);
</script>
```

### Cannot Be Used in Non-Module Scripts

```html
<!-- Does NOT work -->
<script>
  const data = await fetch('/api'); // SyntaxError
</script>
```

## Potential Pitfalls and Performance Considerations

### Blocking Dependent Modules

```javascript
// heavy.mjs — downloads 5MB of data on startup
const bigData = await fetch('/large-dataset.json').then(r => r.json());
export { bigData };

// app.mjs — blocked until heavy.mjs finishes
import { bigData } from './heavy.mjs';
// App cannot start until heavy.mjs completes
```

If `heavy.mjs` is slow, the entire import chain is delayed. Use lazy imports to avoid blocking the initial render:

```javascript
// Better: lazy import
async function getHeavyData() {
  const { bigData } = await import('./heavy.mjs');
  return bigData;
}
```

### Deadlock Risk

If two modules await each other's exports, you have a circular async dependency deadlock:

```javascript
// a.mjs
import { b } from './b.mjs'; // Waits for b.mjs
export const a = await Promise.resolve(b + 1);

// b.mjs
import { a } from './a.mjs'; // Waits for a.mjs — DEADLOCK
export const b = await Promise.resolve(a + 1);
```

The JavaScript engine will detect this as a "module dependency cycle" and the result is undefined behavior or a deadlock depending on the runtime. Avoid circular async dependencies.

### Unhandled Rejections Block Module Loading

```javascript
// broken.mjs
const data = await fetch('/api/that-fails'); // Network error
export const value = await data.json();

// Importer will see:
// import { value } from './broken.mjs'; — This rejects!
```

Any rejection inside a TLA module propagates to the importing module as a dynamic import failure. Handle errors explicitly:

```javascript
// safe.mjs
let config;
try {
  const res = await fetch('/api/config');
  config = await res.json();
} catch (err) {
  console.warn('Config load failed, using defaults:', err);
  config = DEFAULT_CONFIG;
}

export { config };
```

## Node.js Considerations

### Enabling Top-Level Await in Node.js

In Node.js (v14.8+ experimentally, v16+ stable), TLA works in:

- Files with `.mjs` extension
- Files with `.js` extension in a package with `"type": "module"` in `package.json`
- `--input-type=module` flag for stdin/eval

```json
// package.json
{
  "type": "module"
}
```

```javascript
// server.mjs
const { createServer } = await import('node:http'); // Works!
```

### CommonJS Interop

CommonJS modules cannot use TLA and cannot synchronously `require()` ES modules that use TLA (because CommonJS `require` is synchronous). Use dynamic `import()` instead:

```javascript
// CommonJS file
async function main() {
  const { config } = await import('./config.mjs');
  startServer(config);
}
main();
```

## Comparison Table

| Feature | IIFE async | Exported init() | Top-Level Await |
|---------|-----------|----------------|----------------|
| Initialization guarantee | No | Requires manual call | Yes |
| Error propagation | Local only | Caller responsibility | Module load failure |
| Blocks importers | No | No | Yes (by design) |
| Clean API | No | No | Yes |
| Works in CJS | Yes | Yes | No |
| Runtime support | All | All | ES Modules only |
| Circular dep risk | No | No | Yes |

## Best Practices

### 1. Keep Awaits at the Top Level Short and Essential

TLA should be reserved for truly necessary initialization, not general async work. Long-running operations at module level degrade startup time:

```javascript
// Good — quick config load required by all module code
const config = await loadConfig();

// Bad — this should be lazy
const allUsers = await db.query('SELECT * FROM users'); // Not needed at startup
```

### 2. Handle Errors to Avoid Invisible Module Load Failures

Always wrap TLA in try/catch when the async operation might fail and you want a graceful fallback. Silent module load failures are hard to debug.

### 3. Use Dynamic import() for Optional or Conditional Loading

For feature detection and optional modules, dynamic import inside TLA is more appropriate than static import:

```javascript
const module = await import('./optional-feature.mjs').catch(() => null);
if (module) {
  module.initialize();
}
```

### 4. Be Aware of the Impact on Bundle Splitting

Bundlers like webpack and Rollup have varying support for TLA and may emit async chunks. Verify your bundler's behavior before relying on TLA in production:

```javascript
// Rollup/webpack handle TLA but emit async module wrappers
// Verify output and check browser compatibility
```

### 5. Prefer Static Initialization for Synchronous Operations

If your initialization is synchronous, don't use TLA — it adds unnecessary complexity to the module graph:

```javascript
// No TLA needed — synchronous
export const PI = Math.PI;
export const VERSION = '1.2.3';
```

## Interview Questions

### Q1: What is top-level await and what problem does it solve?

**Answer:** Top-level await allows `await` to be used at the module scope (outside of any async function) in ES modules. It solves the problem of async module initialization — previously, code that needed to fetch data or establish connections at startup required async IIFE wrappers that provided no initialization guarantee to importers. With TLA, the module system itself handles async initialization: modules that import a TLA module automatically wait for it to fully initialize before their own bodies execute.

### Q2: Does top-level await block the main thread?

**Answer:** No. TLA is integrated into the module loading algorithm, not the synchronous execution model. While waiting for an async operation, the JavaScript thread is free to evaluate sibling modules and other tasks. Only modules that transitively import the awaiting module are blocked from executing their own bodies. The host environment (browser or Node.js) awaits the entire module graph asynchronously.

### Q3: Can top-level await be used in CommonJS modules?

**Answer:** No. TLA is only valid in ES modules. CommonJS uses synchronous `require()`, and the CommonJS module system has no mechanism to handle async evaluation. Using `await` at the top level of a `.js` file that is evaluated as CommonJS will throw a `SyntaxError`. The file must be treated as an ES module (`.mjs` extension or `"type": "module"` in `package.json`) for TLA to work.

### Q4: What happens if a top-level await expression rejects?

**Answer:** The module fails to load, and any `import` statement or dynamic `import()` call that depends on it rejects as well. This propagates up the import chain. Modules that catch the dynamic import failure (via `.catch()` or try/catch around `await import()`) can recover; static `import` failures that are not caught will prevent the importing module from executing.

### Q5: Can TLA cause deadlocks?

**Answer:** Yes, in circular dependency scenarios. If module A has top-level await that depends on an export from module B, and module B has top-level await that depends on an export from module A, neither module can complete initialization. The specification defines deterministic behavior for this case based on module graph traversal order, but the result is that at least one module will receive `undefined` for the awaited value, causing subtle bugs. Avoid circular dependencies in async module graphs.

## Common Pitfalls

### 1. Using TLA in CommonJS Context

```javascript
// File evaluated as CommonJS (no "type": "module")
const data = await fetch('/api'); // SyntaxError: Unexpected reserved word
```

Ensure your file is recognized as an ES module.

### 2. Forgetting That Importers Are Blocked

```javascript
// slow.mjs — takes 5 seconds
const data = await longFetch();
export { data };

// app.mjs — doesn't need data on startup but imports slow.mjs
import { data } from './slow.mjs'; // Blocked for 5 seconds!
import { render } from './ui.mjs'; // Also blocked — must wait for all imports
```

Use dynamic import to avoid blocking:

```javascript
// app.mjs
import { render } from './ui.mjs'; // Not blocked

render(); // Renders immediately

// Load data lazily when needed
button.onclick = async () => {
  const { data } = await import('./slow.mjs');
  renderData(data);
};
```

### 3. Not Handling Async Initialization Errors

```javascript
// Risky — rejection crashes module loading with no useful message
export const db = await connectToDb(); // Might throw!
```

Always add error handling:

```javascript
let db;
try {
  db = await connectToDb();
} catch (err) {
  console.error('Database connection failed:', err);
  db = createInMemoryFallback();
}
export { db };
```

## Resources

### Official Documentation
- [MDN: Top-level await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await#top_level_await)
- [V8: Top-level await](https://v8.dev/features/top-level-await)
- [TC39: Top-level await proposal](https://github.com/tc39/proposal-top-level-await)
- [Node.js: ECMAScript modules — Top-level await](https://nodejs.org/api/esm.html#top-level-await)

### Articles and Guides
- [Using ES modules in Node.js](https://nodejs.org/api/esm.html) — Official Node.js documentation
- [JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) — MDN guide
- [Webpack: Top Level Await](https://webpack.js.org/configuration/experiments/#experimentstopleavelaiting) — Bundler support
