# ESM vs CommonJS (CJS)

## The Idea

**In plain English:** JavaScript has two different systems for splitting code into separate files and sharing pieces between them — ESM (the modern standard) and CommonJS (the older Node.js way). Think of them as two different sets of rules for how files talk to each other.

**Real-world analogy:** Imagine a school library with two borrowing systems. The old system (CommonJS) lets you walk in at any moment, ask for any book on the spot, and you get a photocopy of the page you need — changes to the original later don't affect your copy. The new system (ESM) requires you to hand in a full request list before the library opens, gets everything ready in advance, and gives you a live window into the shelf — if the book is updated, you see the new version automatically.

- The library request list submitted before opening = ESM `import` statements (declared upfront, analyzed before code runs)
- Walking in anytime and asking on the spot = CJS `require()` (can be called anywhere, any time, even inside an `if` statement)
- Getting a photocopy of a page = CJS value copy (you get the value as it was at that moment, not a live link)
- A live window into the shelf = ESM live binding (your imported variable reflects any future changes to the exported value)

---

## The Two Module Systems

**CommonJS (CJS)** — Node.js's original module system. Synchronous, dynamic, designed for server environments.

**ESM (ES Modules)** — The official JavaScript standard (ES2015). Asynchronous, static, works in both browsers and Node.js.

---

## Syntax Comparison

```js
// CommonJS
const path = require('path');
const { join } = require('path');
module.exports = { myFunc };
module.exports.namedExport = value;

// ESM
import path from 'path';
import { join } from 'path';
export { myFunc };
export const namedExport = value;
export default myFunc;
```

---

## Key Differences

### 1. Static vs Dynamic

ESM imports are **static** — evaluated at parse time, before code executes. CJS requires are **dynamic** — evaluated at runtime.

```js
// ✅ ESM: must be at top level, cannot be conditional
import { something } from './module.js';

// ❌ ESM: this is NOT valid
if (condition) {
  import { something } from './module.js'; // syntax error
}

// ✅ CJS: require anywhere, conditionally
if (condition) {
  const { something } = require('./module.js'); // valid
}
```

This is why ESM enables tree shaking (bundlers can statically analyze what's used) while CJS cannot be reliably tree-shaken.

### 2. Live Bindings vs Snapshots

ESM exports are **live bindings** — if the exported value changes, the importer sees the change:

```js
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
console.log(count); // 1 ← sees the live update!
```

CJS exports are **value copies** (snapshots at require time):

```js
// counter.js
let count = 0;
module.exports = { count, increment: () => count++ };

// main.js
const { count, increment } = require('./counter.js');
console.log(count); // 0
increment();
console.log(count); // 0 ← still 0! count was copied by value
```

### 3. `this` at the Top Level

- CJS: `this` at top level refers to `module.exports`
- ESM: `this` at top level is `undefined`

### 4. `__dirname` / `__filename`

```js
// CJS: available by default
console.log(__dirname);   // /Users/me/project/src
console.log(__filename);  // /Users/me/project/src/file.js

// ESM: not available — use import.meta instead
import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

### 5. Dynamic Import

Both systems support dynamic imports, but the syntax differs:

```js
// ESM dynamic import (returns a Promise)
const module = await import('./module.js');

// CJS require (synchronous)
const module = require('./module.js');
```

---

## Interop: Mixing CJS and ESM

```js
// ESM importing CJS: works (CJS is treated as a default export)
import cjsModule from './legacy.cjs';
const { func } = cjsModule;

// CJS importing ESM: CANNOT use require() for ESM files
// Must use dynamic import()
const esmModule = await import('./modern.mjs');
```

---

## File Extensions and package.json

```json
// package.json — declare module type
{ "type": "module" }    // all .js files treated as ESM
{ "type": "commonjs" }  // all .js files treated as CJS (default)

// Use explicit extensions to override:
// .mjs → always ESM
// .cjs → always CJS
```

---

## In Bundlers (Webpack/Vite)

Bundlers like Webpack and Vite handle both transparently. They:
- Transform CJS to their internal module format
- Process ESM statically for tree shaking
- Allow mixing both in the same project

---

## Common Interview Questions

**Q: Why does ESM enable tree shaking but CJS doesn't?**
ESM imports are static and analyzed at parse time — bundlers know exactly which exports are used. CJS `require()` is dynamic and can be conditional, computed, or depend on runtime values — bundlers can't safely eliminate "unused" exports.

**Q: Can you use `await` at the top level of an ESM module?**
Yes — top-level `await` is a valid ESM feature. `await fetch(url)` at the module top level is valid. It is NOT valid in CJS (CommonJS doesn't understand `await` at all).

**Q: What does `"type": "module"` in package.json do?**
Makes Node.js treat all `.js` files in that package as ESM. Without it, `.js` files are CJS by default. Use `.mjs`/`.cjs` extensions to override on a per-file basis regardless of the package type.
