# ESM vs CommonJS: Module System Semantics and Live Bindings

## Overview

JavaScript has two dominant module systems: CommonJS (CJS), which became the standard for Node.js in 2009, and ECMAScript Modules (ESM), standardized in ES2015 and now fully supported in both browsers and Node.js. Understanding the differences in their semantics — particularly how they resolve dependencies, when they execute, and how they expose bindings — is critical for senior engineers dealing with bundlers, dual-format packages, tree shaking, and interoperability edge cases.

## CommonJS: The Original Node.js Module System

### How CommonJS Works

CommonJS modules are loaded synchronously. When Node.js encounters `require()`, it reads the file, evaluates it immediately, and returns the `module.exports` object. The key characteristic is that `require()` returns a **value** — a snapshot of whatever `module.exports` was at the time the module finished executing.

```javascript
// math.cjs
let count = 0;

function increment() {
  count++;
}

function getCount() {
  return count;
}

module.exports = { count, increment, getCount };
```

```javascript
// consumer.cjs
const { count, increment, getCount } = require('./math.cjs');

console.log(count);      // 0
increment();
console.log(count);      // 0 — still 0! (copied value)
console.log(getCount()); // 1 — function reference is live
```

### module.exports vs exports

```javascript
// These are equivalent
module.exports.foo = 'bar';
exports.foo = 'bar';

// This DOES NOT work — reassigning exports breaks the reference
exports = { foo: 'bar' }; // exports is now a new object, not module.exports

// Correct way to export a single value
module.exports = function greet(name) {
  return `Hello, ${name}`;
};
```

### Caching and Singletons

CommonJS caches modules after the first `require()`. Subsequent `require()` calls return the cached `module.exports` object.

```javascript
// counter.cjs
let count = 0;
module.exports = {
  increment: () => ++count,
  getCount: () => count
};
```

```javascript
// a.cjs
const counter = require('./counter.cjs');
counter.increment();

// b.cjs
const counter = require('./counter.cjs');
counter.increment();

// main.cjs
require('./a.cjs');
require('./b.cjs');
const counter = require('./counter.cjs');
console.log(counter.getCount()); // 2 — same instance
```

This singleton behavior is often relied upon for shared state like database connections or configuration objects.

### Circular Dependencies in CJS

CommonJS handles circular dependencies by returning an **incomplete** module:

```javascript
// a.cjs
const b = require('./b.cjs');
console.log('a got b.value:', b.value); // undefined — b not done loading

module.exports = { value: 'a' };

// b.cjs
const a = require('./a.cjs');
console.log('b got a.value:', a.value); // 'a' — depends on load order

module.exports = { value: 'b' };
```

The first module to trigger the cycle receives a partially-constructed object. This can cause subtle `undefined` bugs that are hard to diagnose.

## ECMAScript Modules: Static Linking and Live Bindings

### Static Analysis Phase

ESM is fundamentally different from CJS: it is **statically analyzed** before execution. When a JavaScript engine encounters an ES module, it parses the entire import graph first — resolving all modules, allocating bindings, and linking them — before executing any code. This two-phase approach is what enables tree shaking, top-level `await`, and live bindings.

```javascript
// math.mjs
export let count = 0;

export function increment() {
  count++;
}
```

```javascript
// consumer.mjs
import { count, increment } from './math.mjs';

console.log(count); // 0
increment();
console.log(count); // 1 — live binding! reflects the updated value
```

### Live Bindings: The Critical Difference

This is arguably the most important semantic difference between ESM and CJS. ESM exports are **live read-only bindings** to the original variable in the exporting module. They are not copies of values — they are views into the exporting module's namespace.

```javascript
// timer.mjs
export let seconds = 0;

setInterval(() => {
  seconds++;
}, 1000);
```

```javascript
// display.mjs
import { seconds } from './timer.mjs';

setInterval(() => {
  console.log(seconds); // Always reads the current value — 1, 2, 3...
}, 1000);
```

In CJS, destructuring `const { seconds } = require('./timer.cjs')` would capture the value at require-time and never update.

Live bindings are read-only from the consumer's perspective:

```javascript
// consumer.mjs
import { count } from './math.mjs';

count = 10; // TypeError: Assignment to constant variable
// You cannot write to an imported binding
```

### Module Namespace Objects

Importing a namespace gives you a live, read-only view of all exports:

```javascript
import * as math from './math.mjs';

console.log(math.count); // 0
math.increment();
console.log(math.count); // 1 — namespace object reflects live bindings

// Namespace objects are sealed:
math.newProp = 'x'; // TypeError in strict mode
```

### Circular Dependencies in ESM

ESM handles circular dependencies cleanly because bindings are allocated (but not yet assigned values) before execution:

```javascript
// a.mjs
import { b } from './b.mjs';
export const a = 'value-a';

console.log(b); // Works correctly — live binding resolves after both execute

// b.mjs
import { a } from './a.mjs';
export const b = 'value-b';

console.log(a); // Works correctly
```

Because ESM allocates slots for all bindings during the linking phase, by the time any module's body executes, all circular binding slots exist and will be filled.

### Top-Level await

ESM supports top-level `await` — a capability impossible in synchronous CJS:

```javascript
// config.mjs
const response = await fetch('https://api.example.com/config');
export const config = await response.json();
```

```javascript
// app.mjs
import { config } from './config.mjs';
// 'config' is guaranteed to be resolved before this code runs
console.log(config.apiKey);
```

Top-level `await` makes the entire importing module async-aware. Any module that imports from a top-level-await module will wait for it to settle.

## File Extensions and package.json Type

Node.js determines module format from file extension or `package.json`:

```json
// package.json — declare entire package as ESM
{
  "type": "module"
}
```

```javascript
// With "type": "module", .js files are ESM
// Use .cjs for CommonJS files

// Without "type": "module" (default), .js files are CJS
// Use .mjs for ESM files
```

```bash
# Extension rules
# .mjs  — always ESM
# .cjs  — always CJS
# .js   — depends on nearest package.json "type" field
```

## Interoperability: Using CJS from ESM and Vice Versa

### ESM importing CJS

```javascript
// default import works (entire module.exports)
import lodash from 'lodash'; // lodash is CJS, this works

// Named imports from CJS may or may not work depending on the bundler/runtime
// Node.js does NOT support named static imports from CJS
import { cloneDeep } from 'lodash'; // Works in Webpack/Vite, NOT in native Node ESM

// Node.js workaround: destructure after default import
import lodash from 'lodash';
const { cloneDeep } = lodash;
```

### CJS importing ESM

```javascript
// require() cannot load ESM — will throw ERR_REQUIRE_ESM
// You must use dynamic import()
async function loadESModule() {
  const { default: myESModule } = await import('./esm-module.mjs');
  return myESModule;
}
```

This is an important architectural constraint: CJS cannot synchronously consume ESM.

## Comparison Table

| Feature | CommonJS (CJS) | ESM |
|---|---|---|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | Synchronous, runtime | Asynchronous, static |
| Analysis | Dynamic (runtime) | Static (parse time) |
| Exports | Copied values | Live bindings |
| Circular deps | Partial object returned | Live bindings handle correctly |
| Top-level await | No | Yes |
| Tree shaking | Not possible | Possible (static analysis) |
| Browser native | No | Yes |
| Node.js native | Yes (default) | Yes (`"type":"module"` or `.mjs`) |
| Cache behavior | Singleton via require cache | Module instance per realm |
| Dynamic loading | `require()` anywhere | `import()` (dynamic) |
| Conditional imports | Easy (if/else around require) | `import()` needed |

## Best Practices

### 1. Prefer ESM for New Projects

ESM is the standard and enables tree shaking, top-level await, and better static analysis. Use it by default:

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

### 2. Publish Dual-Format Packages

For libraries consumed by both CJS and ESM users, ship both formats:

```bash
# Using a build tool like tsup or Rollup
tsup src/index.ts --format cjs,esm --dts
```

### 3. Avoid Relying on Module Side Effects for CJS Singletons

The CJS singleton pattern is fragile. If a module is bundled separately or accessed from different `require()` caches, you lose the singleton guarantee:

```javascript
// Prefer dependency injection or explicit singletons
// rather than relying on require() cache
```

### 4. Understand Live Binding Implications

When re-exporting, live bindings flow through:

```javascript
// re-export.mjs
export { count, increment } from './math.mjs';

// consumer.mjs
import { count } from './re-export.mjs';
// 'count' is STILL a live binding to math.mjs's 'count'
```

### 5. Use Named Exports for Tree Shaking

Default exports prevent dead code elimination in many bundlers:

```javascript
// Bad for tree shaking
export default { foo, bar, baz };

// Good for tree shaking
export { foo, bar, baz };
```

---

## Supplementary: Import Maps

Import maps let you control how bare module specifiers resolve in the browser — without a bundler. They're primarily useful for CDN-based development, import aliasing, and polyfilling built-in modules.

### Basic Usage

```html
<!-- Must appear before any module scripts -->
<script type="importmap">
{
  "imports": {
    "lodash":       "https://cdn.jsdelivr.net/npm/lodash-es@4/lodash.js",
    "lodash/":      "https://cdn.jsdelivr.net/npm/lodash-es@4/",
    "@/":           "/src/",
    "react":        "/node_modules/react/index.js",
    "react-dom":    "/node_modules/react-dom/index.js"
  }
}
</script>

<script type="module">
  import _ from 'lodash';                    // → CDN URL
  import chunk from 'lodash/chunk.js';       // → CDN URL (trailing slash maps)
  import Button from '@/components/Button.js'; // → /src/components/Button.js
</script>
```

### Scoped Mappings

Different parts of the app can resolve the same specifier differently:

```json
{
  "imports": {
    "moment": "/moment-v2.js"
  },
  "scopes": {
    "/legacy/": {
      "moment": "/moment-v1.js"
    }
  }
}
```

### Use Cases

```html
<!-- Dev: use source files; Prod: swap to minified without changing code -->
<script type="importmap">
{
  "imports": {
    "react": "/vendor/react.development.js"
  }
}
</script>

<!-- Test environments: swap real modules for mocks -->
<script type="importmap">
{
  "imports": {
    "./analytics.js": "./analytics.mock.js"
  }
}
</script>
```

### Limitations
- Only one `<script type="importmap">` allowed per page (must appear before module scripts)
- No dynamic import maps at runtime
- Not supported in older browsers (use `es-module-shims` polyfill)
- Bundlers (Vite, Webpack) use their own resolution and ignore import maps

### Supplementary: `scheduler.postTask()` — Fine-Grained Task Scheduling

The Scheduler API lets you assign explicit priorities to tasks, replacing ad-hoc `setTimeout(fn, 0)` hacks:

```javascript
// Priority levels: 'user-blocking' | 'user-visible' (default) | 'background'

// Critical UI update — runs before other queued tasks
await scheduler.postTask(() => updateCart(items), {
  priority: 'user-blocking',
});

// Non-critical analytics — deferred to idle time
scheduler.postTask(() => sendAnalytics(event), {
  priority: 'background',
});

// Cancel a task
const controller = new TaskController({ priority: 'background' });
const task = scheduler.postTask(() => heavyWork(), {
  signal: controller.signal,
});
controller.abort(); // cancels if not started yet

// Change priority dynamically
const ctrl = new TaskController({ priority: 'background' });
scheduler.postTask(doWork, { signal: ctrl.signal });
ctrl.setPriority('user-blocking'); // promote when user interacts

// yield() — break a Long Task into chunks, yielding between each
async function processLargeList(items) {
  for (let i = 0; i < items.length; i++) {
    if (i % 50 === 0) await scheduler.yield(); // yield to browser between chunks
    processItem(items[i]);
  }
}
```

`scheduler.yield()` is the modern replacement for `await new Promise(r => setTimeout(r, 0))` — it yields control back to the browser (allowing user interactions and paints) then resumes at the same priority.

---

## Interview Questions

### Q1: What is a live binding and how does it differ from CJS module exports?

**Answer:** An ESM live binding is a read-only reference to the binding in the exporting module's namespace. When the exporting module mutates the variable, all importers see the updated value. In contrast, CJS `module.exports` exports a snapshot: when you `const { count } = require('./counter')`, you receive the value of `count` at that moment — a copy. Subsequent changes to the original `count` in the exporting module are not reflected.

### Q2: Why can't you use `require()` to import an ESM module in Node.js?

**Answer:** CommonJS `require()` is synchronous — it blocks the event loop until the module fully loads. ESM supports top-level `await`, which means ESM modules may be asynchronous. A synchronous API cannot accommodate an asynchronous initialization phase. Node.js therefore forbids `require()` of `.mjs` files and throws `ERR_REQUIRE_ESM`. The solution is to use dynamic `import()`, which returns a Promise and properly handles async module initialization.

### Q3: How does ESM enable tree shaking but CJS does not?

**Answer:** ESM import and export statements are statically analyzable — they must appear at the top level, cannot be inside conditionals or functions, and their names are resolved at parse time before execution. A bundler can therefore build a complete dependency graph and determine which exports are actually used. CJS `require()` is a runtime function call that can appear anywhere, take computed arguments, and be conditionally executed, so a bundler cannot statically determine which exports are consumed without running the code.

### Q4: What happens with circular dependencies in ESM vs CJS?

**Answer:** In CJS, when module A requires B and B requires A, whichever module triggers the cycle gets an incomplete `module.exports` object from the other — any properties added after that point are invisible. In ESM, the engine allocates all binding slots during the static linking phase before any code runs. When execution begins, all modules in the cycle know where every imported binding will live. As long as you don't access a circular binding before the exporting module's body has executed (i.e., avoid accessing at module initialization time), circular imports work correctly because the binding slot will be populated by the time it's accessed.

### Q5: What does `"type": "module"` in package.json actually change?

**Answer:** It tells Node.js to treat all `.js` files in that package as ESM. Without it (the default), `.js` files are CJS. The explicit extension overrides still apply: `.mjs` is always ESM and `.cjs` is always CJS regardless of the `type` field. This field also affects how bundlers and tools like TypeScript interpret module format defaults for the package.

## Common Pitfalls

### 1. Destructuring CJS as if it Were ESM Live Bindings

```javascript
// CJS — count is a copied value, NOT a live binding
const { count } = require('./counter.cjs');
// count will never update even if counter module increments it
// Use getCount() function instead to observe live state
```

### 2. Named Imports from CJS in Node.js ESM

```javascript
// This works in webpack/vite but FAILS in native Node.js ESM
import { cloneDeep } from 'lodash'; // ERR_NAMED_IMPORT in Node

// Correct Node.js approach:
import lodash from 'lodash';
const { cloneDeep } = lodash;
```

### 3. Forgetting That CJS require() Is Not Lazy for Types

```javascript
// Both of these execute math.cjs immediately
const math1 = require('./math.cjs');
const math2 = require('./math.cjs'); // returns cache, does not re-execute

// But the first one is NOT lazy — unlike dynamic import()
```

### 4. Mutating Named Exports From Within the Module

```javascript
// utils.mjs
export let config = {};

export function setConfig(value) {
  config = value; // This is fine — module can reassign its own binding
}
```

```javascript
// consumer.mjs
import { config, setConfig } from './utils.mjs';

setConfig({ debug: true });
console.log(config); // { debug: true } — live binding updated
config = {}; // TypeError — consumer cannot reassign
```

### 5. Mixing Default and Named Exports Inconsistently

```javascript
// Hard for consumers to use correctly
export default class Foo {}
export const helper = () => {};

// Better: consistent named exports
export class Foo {}
export const helper = () => {};
```

## Resources

### Official Documentation
- [MDN: JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)
- [Node.js: ECMAScript modules](https://nodejs.org/api/esm.html)
- [Node.js: CommonJS modules](https://nodejs.org/api/modules.html)
- [TC39 Modules Specification](https://tc39.es/ecma262/#sec-modules)

### Articles and Guides
- [CommonJS vs ESM — What's the difference?](https://blog.logrocket.com/commonjs-vs-es-modules-node-js/)
- [The state of ES modules in Node.js](https://blog.risingstack.com/using-es-modules-esm-in-node-js/)
- [How ESM live bindings work under the hood](https://hacks.mozilla.org/2018/03/es-modules-a-cartoon-deep-dive/)

### Tools
- [tsup](https://tsup.egoist.dev/): Zero-config dual-format bundler
- [Rollup](https://rollupjs.org/): Module bundler with excellent ESM support
- [esm-check](https://github.com/nicolo-ribaudo/esm-check): Detects CJS/ESM interop issues
