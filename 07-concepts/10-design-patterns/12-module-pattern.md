# Module Pattern

## The Idea

**In plain English:** A module is a self-contained package of code that keeps some things private (hidden inside) and only shares what it chooses to share with the outside world. Think of it as a vending machine for code: you can use the buttons on the front, but the internal wiring and inventory are locked away where you cannot touch them.

**Real-world analogy:** Imagine a vending machine in a school hallway. You press a button to get a snack, check the display to see prices, and collect what comes out — but you cannot reach inside to grab items directly, change the prices yourself, or rewire the coin slot.

- The locked cabinet = the private variables and helper functions hidden inside a module
- The buttons and display on the front = the exported functions and values the module lets you use
- The machine only dispensing one item at a time = controlled access that prevents outsiders from corrupting internal state

---

## Overview

The Module pattern encapsulates related code — data, functions, and state — behind a public interface, hiding implementation details. It provides namespace isolation, preventing global scope pollution, and enables controlled exposure of functionality. JavaScript has evolved through several module formats: IIFE-based modules (pre-ES6), revealing module pattern, CommonJS (Node.js), AMD, and finally native ES Modules (ESM). Understanding the evolution clarifies why ES Modules are the standard today.

## The Problem: Global Namespace Pollution

```javascript
// ❌ Pre-module era — everything in global scope
var counter = 0;
var increment = function() { counter++; };
var reset = function() { counter = 0; };
// counter, increment, reset all pollute window.*
// Any script can accidentally overwrite counter = 'oops'
```

## IIFE Module (Immediately Invoked Function Expression)

The original JavaScript module pattern — a function that executes immediately and returns a public API:

```javascript
// IIFE Module
var Counter = (function() {
  // Private — not accessible outside the IIFE
  var count = 0;
  var MAX = 100;

  function clamp(n) {
    return Math.min(Math.max(n, 0), MAX);
  }

  // Public API — returned object
  return {
    increment: function() { count = clamp(count + 1); },
    decrement: function() { count = clamp(count - 1); },
    reset: function() { count = 0; },
    getCount: function() { return count; },
  };
})();

Counter.increment();
Counter.increment();
console.log(Counter.getCount()); // 2
console.log(Counter.count);      // undefined — private!
```

### Augmenting IIFE Modules

```javascript
// Extend a module across multiple files
var Module = Module || {};  // create or reuse

Module.Part1 = (function(mod) {
  mod.feature1 = function() { /* ... */ };
  return mod;
})(Module);

Module.Part2 = (function(mod) {
  mod.feature2 = function() { /* ... */ };
  return mod;
})(Module);
```

## Revealing Module Pattern

Exposes only specific functions by name — keeps all implementation private, reveals selectively:

```javascript
// Revealing Module Pattern
var Cart = (function() {
  // Private
  var items = [];
  var TAX_RATE = 0.1;

  function calculateTax(amount) {
    return amount * TAX_RATE;
  }

  function getSubtotal() {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  // Public functions (but defined privately above)
  function addItem(product, quantity) {
    var existing = items.find(i => i.id === product.id);
    if (existing) {
      existing.quantity += quantity;
    } else {
      items.push({ ...product, quantity });
    }
  }

  function removeItem(productId) {
    items = items.filter(i => i.id !== productId);
  }

  function getTotal() {
    var subtotal = getSubtotal();
    return subtotal + calculateTax(subtotal);
  }

  function getItems() {
    return [...items]; // return copy — callers can't mutate internal array
  }

  // Reveal only the public interface
  return {
    add: addItem,
    remove: removeItem,
    total: getTotal,
    items: getItems,
  };
  // calculateTax and getSubtotal are NOT revealed — private helpers
})();

Cart.add({ id: '1', name: 'Widget', price: 10 }, 2);
console.log(Cart.total()); // 22 (20 + 2 tax)
Cart.calculateTax(100);    // TypeError: Cart.calculateTax is not a function
```

## CommonJS (Node.js)

```javascript
// math.js — CommonJS module (Node.js)
const PI = Math.PI;

function circleArea(radius) {
  return PI * radius ** 2;
}

function circlePerimeter(radius) {
  return 2 * PI * radius;
}

// Explicit exports
module.exports = {
  circleArea,
  circlePerimeter,
};

// Or: export individually
exports.circleArea = circleArea;

// consumer.js
const { circleArea, circlePerimeter } = require('./math');
// or: const math = require('./math');
```

**CJS characteristics:**
- Synchronous `require()` (fine for Node.js; problematic for browsers)
- Dynamic — `require()` can be called anywhere, conditionally
- NOT tree-shakeable (dynamic evaluation)
- module caching — `require()` returns cached module after first load

## ES Modules (ESM) — The Standard

```typescript
// math.ts — ES Module
export const PI = Math.PI;

export function circleArea(radius: number): number {
  return PI * radius ** 2;
}

export function circlePerimeter(radius: number): number {
  return 2 * PI * radius;
}

// Default export (one per module)
export default function formatArea(area: number): string {
  return `${area.toFixed(2)} sq units`;
}

// consumer.ts
import { circleArea, circlePerimeter } from './math';
import formatArea from './math';  // default import

// Namespace import
import * as MathUtils from './math';
MathUtils.circleArea(5);

// Re-export (barrel file)
export { circleArea, circlePerimeter } from './math';
export { default as formatArea } from './math';
```

**ESM characteristics:**
- Static structure — imports and exports at top level only
- Asynchronous loading (native browser support)
- Tree-shakeable (static analysis possible)
- Live bindings — imported values reflect mutations in the exporting module
- Module scope — no global pollution

### Live Bindings (ESM-specific)

```typescript
// counter.ts
export let count = 0;
export function increment() { count++; }

// consumer.ts
import { count, increment } from './counter';
console.log(count);   // 0
increment();
console.log(count);   // 1 ← live binding! CJS would still show 0
// In CJS, { count } would be a copy of the value at import time
```

## Module Scope as Module Pattern

In modern development with ESM, the module scope itself IS the module pattern:

```typescript
// store.ts — module scope provides encapsulation
// Private state — not exported; inaccessible to consumers
let _state: AppState = initialState;
const _subscribers = new Set<Subscriber>();

// Private function — not exported
function notifySubscribers(): void {
  _subscribers.forEach(sub => sub(_state));
}

// Public API — exported functions
export function getState(): AppState {
  return { ..._state }; // return copy
}

export function dispatch(action: Action): void {
  _state = reducer(_state, action);
  notifySubscribers();
}

export function subscribe(fn: Subscriber): () => void {
  _subscribers.add(fn);
  return () => _subscribers.delete(fn); // unsubscribe
}

// consumers.ts
import { getState, dispatch, subscribe } from './store';
// _state and _subscribers are invisible — true module encapsulation
```

## Dynamic Imports (Code Splitting)

ES Modules support dynamic `import()` — returns a Promise:

```typescript
// Static import — always loaded
import { HeavyChart } from './HeavyChart';

// Dynamic import — loaded only when needed
async function loadDashboard() {
  const { HeavyChart } = await import('./HeavyChart');
  // HeavyChart loaded only when this function runs
}

// React.lazy (dynamic import for components)
const AdminPanel = React.lazy(() => import('./AdminPanel'));
const HeavyEditor = React.lazy(() =>
  import('./HeavyEditor').then(module => ({ default: module.HeavyEditor }))
);

// Conditional loading based on feature flag
async function loadFeature(flagEnabled: boolean) {
  if (!flagEnabled) return;
  const { Feature } = await import('./ExperimentalFeature');
  return Feature;
}

// Route-based splitting
const routes = [
  {
    path: '/dashboard',
    component: React.lazy(() => import('./pages/Dashboard')),
  },
  {
    path: '/admin',
    component: React.lazy(() => import('./pages/Admin')),
  },
];
```

## Module Singleton Pattern

A module is loaded only once — imports from multiple files get the same module instance:

```typescript
// singleton.ts — module is a singleton automatically
class DatabasePool {
  private static connection: Connection;

  static async getConnection(): Promise<Connection> {
    if (!DatabasePool.connection) {
      DatabasePool.connection = await createConnection(config);
    }
    return DatabasePool.connection;
  }
}

// Module-level singleton (simpler)
let _pool: ConnectionPool | null = null;

export async function getPool(): Promise<ConnectionPool> {
  if (!_pool) {
    _pool = await createPool(config);
  }
  return _pool;
}

// Every file that imports getPool gets the same _pool
// because the module is only evaluated once
```

## Namespace Module (TypeScript)

TypeScript namespaces are an older approach to grouping:

```typescript
// ❌ TypeScript namespace — legacy, avoid in modern ESM codebases
namespace MathUtils {
  export function add(a: number, b: number): number { return a + b; }
  export function subtract(a: number, b: number): number { return a - b; }
}

MathUtils.add(1, 2);

// ✅ ESM module — tree-shakeable, standard
// math-utils.ts
export function add(a: number, b: number): number { return a + b; }
export function subtract(a: number, b: number): number { return a - b; }
```

## Common Mistakes

### 1. Circular Dependencies

```typescript
// ❌ Circular dependency — a.ts imports b.ts; b.ts imports a.ts
// a.ts
import { b } from './b';
export const a = `a depends on ${b}`;

// b.ts
import { a } from './a';
export const b = `b depends on ${a}`;
// At runtime: a is undefined when b tries to use it (circular resolution order)

// ✅ Extract shared dependency to a third module
// shared.ts
export const shared = 'shared value';
// a.ts: import { shared } from './shared';
// b.ts: import { shared } from './shared';
```

### 2. Polluting Module with Side Effects

```typescript
// ❌ Module has unexpected side effects when imported
// analytics.ts
window.analyticsLoaded = true;          // global mutation on import!
document.addEventListener('click', track); // listener on every import!
fetchUserPreferences();                 // network request on import!

// ✅ Side effects in explicit init functions
export function initAnalytics(): void {
  document.addEventListener('click', track);
  fetchUserPreferences();
}
// Only triggered when the consumer explicitly calls initAnalytics()
```

### 3. Mutating Exported Objects

```typescript
// ❌ Exported object is mutated externally — breaks encapsulation
export const config = { apiUrl: '/api', timeout: 5000 };
// Anywhere: import { config } from './config'; config.timeout = 0; // pollution!

// ✅ Export immutable or return copies
export const config = Object.freeze({ apiUrl: '/api', timeout: 5000 });
// Or: export getter
export function getConfig() { return { ...config }; }
```

## Interview Questions

### 1. What is the difference between IIFE modules and ES Modules?

**Answer:** IIFE modules are a pre-language pattern — a function executes immediately and returns a public API object, using closure scope for privacy. They're synchronous, live in the global scope (the returned object), and require manual dependency management. ES Modules are a native language feature — `import`/`export` syntax at the top level, loaded asynchronously by the browser or bundler, with module scope separate from global scope. ESM has static structure (imports must be top-level, not conditional), enabling tree shaking. IIFE modules are dynamically evaluated, making dead code elimination impossible. ESM is the standard for all modern JavaScript.

### 2. What does "live bindings" mean in ES Modules and how does it differ from CommonJS?

**Answer:** In ES Modules, named imports are live bindings — they reflect the current value of the exported variable, including mutations. If a module exports `let count = 0` and later increments it, an importing module sees the updated value. In CommonJS, `const { count } = require('./counter')` copies the value at import time — changes to `count` in the source module are invisible to the importer (the copy has the old value). Live bindings enable patterns like reactive state in modules, but they also mean exports shouldn't be mutated unexpectedly since all importers see the change.

### 3. How does module scope act as a module pattern in modern JavaScript?

**Answer:** In ESM, every file has its own module scope — variables declared in a module are not accessible outside unless explicitly exported. Non-exported declarations are the equivalent of "private." This eliminates the need for IIFE wrappers — the module format itself provides encapsulation. You get a clean separation between the private implementation (non-exported code) and the public interface (exports). A Redux-style store module can declare `let _state` as module-private state and export `dispatch`, `getState`, and `subscribe` as the public API — callers cannot access `_state` directly. This is idiomatic modern JavaScript without any additional pattern.

### 4. When would you use a dynamic import() instead of a static import?

**Answer:** Dynamic imports for: (1) code splitting — routes, heavy components, features only loaded when needed (`React.lazy`); (2) conditional loading based on feature flags or user actions; (3) avoiding loading a module until the user actually needs it (a PDF export library loaded only when the user clicks "Export"); (4) polyfills only loaded on browsers that need them. Static imports for: everything loaded on the initial page. The distinction is: static imports are part of the module graph evaluated at startup; dynamic imports are asynchronous chunks fetched later. Bundlers (Webpack, Vite) split the bundle at dynamic import boundaries, creating separate chunks.
