# Dynamic import() and Code Splitting

## Overview

Dynamic `import()` is a JavaScript expression — not a statement — that loads a module asynchronously at runtime and returns a Promise. Introduced as part of the ECMAScript 2020 specification (though supported by major bundlers years earlier), it is the foundation of code splitting: the technique of partitioning application code into smaller chunks that load on demand rather than upfront. Understanding `import()` deeply — its semantics, its caveats, and how bundlers transform it — is essential for any senior engineer optimizing web application performance.

## Static vs Dynamic Import

### Static import (ESM)

```javascript
// Top-level only — analyzed at parse time
import { format } from 'date-fns';
import UserDashboard from './UserDashboard.js';
```

Static imports must appear at the top level, cannot be inside blocks, and are resolved before any code runs. The module graph is fully known before execution.

### Dynamic import()

```javascript
// Can appear anywhere — returns a Promise
async function loadChart() {
  const { Chart } = await import('chart.js');
  return new Chart(ctx, options);
}

// Conditional loading
if (userPreferences.darkMode) {
  import('./themes/dark.js').then(module => module.apply());
}

// Inside event handlers
button.addEventListener('click', async () => {
  const { openModal } = await import('./Modal.js');
  openModal();
});
```

## Core Semantics

### Return Value

`import()` returns a Promise that resolves to the **module namespace object** — equivalent to `import * as ns from '...'`.

```javascript
const ns = await import('./math.js');
// ns is { add: [Function], subtract: [Function], default: ... }

// Access named exports
const { add, subtract } = await import('./math.js');

// Access default export
const { default: MyComponent } = await import('./MyComponent.js');
// or equivalently:
const module = await import('./MyComponent.js');
const MyComponent = module.default;
```

### Module Namespace Object Properties

```javascript
// math.js
export const PI = 3.14159;
export function square(x) { return x * x; }
export default function main() { return 'main'; }
```

```javascript
const ns = await import('./math.js');

console.log(ns.PI);       // 3.14159
console.log(ns.square(4)); // 16
console.log(ns.default()); // 'main'
console.log(ns[Symbol.toStringTag]); // 'Module'

// Namespace objects are live — same semantics as static import *
```

### Caching Behavior

Like static imports, `import()` is cached per module URL. Multiple calls with the same specifier return the same namespace object:

```javascript
const ns1 = await import('./utils.js');
const ns2 = await import('./utils.js');

console.log(ns1 === ns2); // true — same module namespace object
```

## Code Splitting with Bundlers

### How Bundlers Detect Split Points

Bundlers like Vite, Webpack, and Rollup treat each dynamic `import()` call as a **split point** — a place where the module graph can be divided into separate output chunks.

```javascript
// Webpack/Vite sees this as a split point:
const module = await import('./HeavyFeature.js');
// HeavyFeature.js and all its unique dependencies become a separate chunk
```

### Entry-Point Splitting vs Route Splitting

```javascript
// React Router v6 with lazy loading
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Analytics = lazy(() => import('./pages/Analytics'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/analytics" element={<Analytics />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

Each route becomes a separate chunk downloaded only when the user navigates there.

### Webpack Magic Comments

Webpack provides special comments inside `import()` to control chunk behavior:

```javascript
// Named chunks — multiple imports share the same chunk
const UserModal = await import(
  /* webpackChunkName: "user-features" */ './UserModal.js'
);
const UserProfile = await import(
  /* webpackChunkName: "user-features" */ './UserProfile.js'
);

// Prefetch — download during browser idle time, triggered early
import(/* webpackPrefetch: true */ './OptionalFeature.js');

// Preload — download in parallel with current chunk (higher priority)
import(/* webpackPreload: true */ './CriticalWidget.js');

// Disable lazy loading (include in main bundle)
import(/* webpackMode: "eager" */ './AlwaysNeeded.js');
```

### Vite Dynamic Import Handling

Vite uses Rollup under the hood for production builds. Dynamic imports are split automatically:

```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        // Custom chunk naming
        chunkFileNames: 'chunks/[name]-[hash].js',
        
        // Manual chunk grouping
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash-es', 'date-fns']
        }
      }
    }
  }
};
```

## Patterns and Use Cases

### Lazy Loading Heavy Libraries

```javascript
// Don't load Moment.js/Chart.js/Monaco at startup
async function renderChart(data) {
  const { Chart, registerables } = await import('chart.js');
  Chart.register(...registerables);
  
  const canvas = document.getElementById('myChart');
  return new Chart(canvas, { type: 'bar', data });
}
```

### Conditional Feature Loading

```javascript
// Load polyfills only where needed
async function setupDatePicker() {
  if (!('showPicker' in HTMLInputElement.prototype)) {
    await import('./polyfills/input-show-picker.js');
  }
  initializeDatePicker();
}

// Load platform-specific code
async function loadShareAPI() {
  if (navigator.share) {
    const { nativeShare } = await import('./share/native.js');
    return nativeShare;
  } else {
    const { fallbackShare } = await import('./share/fallback.js');
    return fallbackShare;
  }
}
```

### Dynamic Module Resolution

```javascript
// Build path from variable (bundler creates a chunk per possible match)
async function loadLocale(locale) {
  const messages = await import(`./locales/${locale}.js`);
  return messages.default;
}

// Warning: bundlers create chunks for all files matching the glob pattern
// Use a map for more control:
const localeMap = {
  'en': () => import('./locales/en.js'),
  'fr': () => import('./locales/fr.js'),
  'de': () => import('./locales/de.js')
};

async function loadLocale(locale) {
  const loader = localeMap[locale] ?? localeMap['en'];
  return (await loader()).default;
}
```

### Preloading for Predicted Navigation

```javascript
// Preload on hover — high confidence user will navigate
function NavLink({ to, children }) {
  const handleMouseEnter = () => {
    // Trigger load before click — typically 100-300ms head start
    import(`./pages/${to}.js`);
  };

  return (
    <a href={to} onMouseEnter={handleMouseEnter}>
      {children}
    </a>
  );
}
```

### Import with Timeout and Error Handling

```javascript
function importWithTimeout(specifier, timeoutMs = 5000) {
  const modulePromise = import(specifier);
  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Import timed out: ${specifier}`)), timeoutMs)
  );
  return Promise.race([modulePromise, timeoutPromise]);
}

async function loadFeature() {
  try {
    const { Feature } = await importWithTimeout('./HeavyFeature.js', 3000);
    return new Feature();
  } catch (err) {
    if (err.message.includes('timed out')) {
      return loadFallbackFeature();
    }
    throw err;
  }
}
```

## Limitations and Constraints

### No Dynamic Specifiers Without Bundler Awareness

```javascript
// Bundler cannot create chunks for fully dynamic strings
const path = getPathFromServer(); // runtime value
await import(path); // Bundler cannot analyze this — treated as external

// Template literals with a static prefix are partially analyzable
const feature = 'editor'; // may be dynamic in practice
await import(`./features/${feature}.js`); // Bundler includes all matching files
```

### Circular Imports Still Apply

Dynamic `import()` participates in the same module instance cache as static imports. Circular dependencies via dynamic imports have the same live-binding semantics as static ESM.

### No Synchronous Access

```javascript
// You cannot synchronously read the result — it's always async
const module = import('./sync-looking.js'); // returns Promise, not module
module.doSomething(); // TypeError — module is a Promise
```

### Module Evaluation Happens Once

```javascript
// Side effects run only once regardless of how many times you import()
let executionCount = 0;

// sideEffect.js
executionCount++; // runs once even with multiple import() calls
export const value = 42;
```

## Comparison Table

| Aspect | Static import | Dynamic import() |
|---|---|---|
| Syntax position | Top-level only | Anywhere |
| Resolution time | Parse/link phase | Runtime |
| Return type | Binding (synchronous) | Promise<ModuleNamespace> |
| Code splitting | No | Yes (split point) |
| Conditional loading | No | Yes |
| Tree shakeable | Yes | Per-chunk |
| Circular handling | Static linking | Same ESM semantics |
| Default export access | `import Foo` | `(await import()).default` |
| Available in CJS | No | Yes (in Node.js) |

## Best Practices

### 1. Colocate Related Lazy Modules

```javascript
// Good: feature boundary is clear
const { UserDashboard } = await import('./features/user/Dashboard.js');
const { UserSettings } = await import('./features/user/Settings.js');

// These might share utilities — bundlers can automatically deduplicate
```

### 2. Use React.lazy() as the Standard React Pattern

```javascript
// React.lazy wraps dynamic import() and integrates with Suspense
const LazyComponent = React.lazy(() => import('./LazyComponent'));
// Must export a default React component
```

### 3. Avoid import() Inside Tight Loops

```javascript
// Bad — creates N network requests, bypasses caching in some environments
items.forEach(async (item) => {
  const { process } = await import('./processor.js');
  process(item);
});

// Good — import once, use repeatedly
const { process } = await import('./processor.js');
items.forEach(item => process(item));
```

### 4. Measure with Performance API

```javascript
async function measuredImport(specifier) {
  const start = performance.now();
  const module = await import(specifier);
  const duration = performance.now() - start;
  
  if (duration > 500) {
    console.warn(`Slow dynamic import: ${specifier} took ${duration.toFixed(0)}ms`);
  }
  
  return module;
}
```

### 5. Provide Loading UI for UX

```javascript
async function loadAndRender() {
  showSpinner();
  try {
    const { render } = await import('./HeavyRenderer.js');
    render();
  } finally {
    hideSpinner();
  }
}
```

## Interview Questions

### Q1: What does dynamic import() return and how do you access the default export?

**Answer:** `import()` returns a Promise that resolves to the module namespace object. This is equivalent to `import * as ns`. To access the default export, you use the `default` property: `const { default: MyClass } = await import('./module.js')`. Named exports are accessed directly: `const { helper } = await import('./utils.js')`.

### Q2: How does a bundler like Webpack determine where to create chunk boundaries?

**Answer:** Webpack creates a new chunk at each `import()` call site (and at each entry point). The imported module and any dependencies that are not already included in the current chunk (or a shared chunk) are emitted as a new chunk file. Webpack then replaces the `import()` call with an async chunk-loading mechanism that fetches the chunk via a script tag or `fetch()` (depending on target). If multiple dynamic imports share common dependencies, Webpack extracts those shared deps into a separate "common chunk" based on the `optimization.splitChunks` configuration.

### Q3: What is the difference between prefetch and preload in Webpack?

**Answer:** Both are resource hints that affect how browsers download chunks. `webpackPreload: true` generates a `<link rel="preload">` tag — the browser downloads this resource in parallel with the current page load at high priority, because it will be needed soon. `webpackPrefetch: true` generates a `<link rel="prefetch">` tag — the browser downloads the resource during idle time at low priority, in anticipation of future navigation. Preload is for resources needed in the current navigation; prefetch is for likely future navigation.

### Q4: Why does using a template literal variable in import() cause bundlers to include all matching files?

**Answer:** Bundlers analyze `import()` arguments statically. When the specifier is a template literal like `./locales/${locale}.js`, the bundler cannot know the runtime value of `locale`, so it includes every file that could match the pattern (all `.js` files in `./locales/`). This creates a chunk for each matching file. This is called "dynamic import with a variable specifier" or "glob import" behavior. To minimize this, either enumerate specific paths explicitly, use a static map object that references explicit `import()` calls, or use Vite's `import.meta.glob()` which makes this behavior explicit.

### Q5: Is `import()` available inside CommonJS modules in Node.js?

**Answer:** Yes. Node.js supports dynamic `import()` inside CJS files as the only way to load ESM from CJS. Since `import()` is always async, the CJS code must use `.then()` or be inside an `async` function. This is the intentional escape hatch for CJS code that needs to consume ESM, since synchronous `require()` of ESM is forbidden.

## Common Pitfalls

### 1. Forgetting .default for Default Exports

```javascript
// Wrong
const MyClass = await import('./MyClass.js');
new MyClass(); // TypeError — MyClass is the namespace object, not the class

// Correct
const { default: MyClass } = await import('./MyClass.js');
new MyClass(); // Works
```

### 2. Not Handling Import Failures

```javascript
// Bad — unhandled promise rejection
const module = import('./optional-feature.js');

// Good
try {
  const { Feature } = await import('./optional-feature.js');
  Feature.init();
} catch (err) {
  console.warn('Optional feature unavailable:', err.message);
}
```

### 3. Creating Waterfalls

```javascript
// Bad — sequential imports create a waterfall (each waits for the previous)
const a = await import('./a.js');
const b = await import('./b.js');
const c = await import('./c.js');

// Good — parallel imports
const [a, b, c] = await Promise.all([
  import('./a.js'),
  import('./b.js'),
  import('./c.js')
]);
```

### 4. Over-Splitting Small Modules

Every chunk has overhead: an HTTP round trip (or HTTP/2 stream), module parsing, and initialization. Splitting tiny modules into individual chunks can hurt performance more than it helps. Use bundler analysis tools to check chunk sizes before over-optimizing.

## Resources

### Official Documentation
- [MDN: import()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import)
- [TC39 Dynamic import proposal](https://github.com/tc39/proposal-dynamic-import)
- [Webpack: Code Splitting](https://webpack.js.org/guides/code-splitting/)
- [Vite: Code Splitting](https://vitejs.dev/guide/features#code-splitting)

### Articles and Guides
- [web.dev: Code splitting](https://web.dev/articles/code-splitting-suspense)
- [Patterns.dev: Dynamic Import](https://www.patterns.dev/vanilla/dynamic-import/)
- [JavaScript.info: Dynamic imports](https://javascript.info/modules-dynamic-imports)

### Tools
- [Webpack Bundle Analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
- [Rollup Bundle Visualizer](https://github.com/btd/rollup-plugin-visualizer)
- [vite-bundle-visualizer](https://github.com/KusStar/vite-bundle-visualizer)
