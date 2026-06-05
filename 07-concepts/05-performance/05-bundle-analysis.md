# Bundle Analysis

## The Idea

**In plain English:** Bundle analysis is the process of looking inside the single compressed file (called a "bundle") that a website sends to your browser, to see exactly which pieces of code are inside it and how much space each piece takes up. A "bundle" is like a packed suitcase — all your website's code squished together — and analysis tells you if you accidentally packed things you don't need.

**Real-world analogy:** Imagine you are packing a suitcase for a weekend trip but you keep stuffing in extra clothes "just in case." At the airport, your bag is overweight and you get charged a fee. A bundle analyzer is like laying every item out on the bed before you pack, so you can spot the bulky winter coat (a huge library like moment.js) you packed for a beach trip, the duplicate socks you packed twice (duplicate dependencies), and the fancy outfit you will never actually wear (unused code).

- The suitcase = the JavaScript bundle sent to the browser
- Each item of clothing = a library or piece of code inside the bundle
- The overweight fee = slow page load time caused by a bloated bundle

---

## Overview

You cannot optimize what you cannot see. Bundle analysis is the process of visualizing what's inside your JavaScript bundles — which libraries, how large each module is, what's duplicated, and what you didn't know was even there. This guide covers the primary analysis tools, how to interpret their output, and what the most common bundle bloat patterns look like and how to fix them.

## Why Bundle Analysis Matters

```
Average JavaScript bundle costs:

Network:   1MB gzipped JS ≈ 2-5s download on 3G mobile
CPU:       1MB parsed JS ≈ 8-12s parse + compile on mid-range phone
Memory:    1MB JS ≈ 5-10MB memory after parse + execution

For every 100KB of JS you remove:
  ~0.5s faster TTI on desktop
  ~1-2s faster TTI on mid-range mobile
```

A bundle analysis session routinely finds 20-60% of a production bundle is either unused code, duplicate dependencies, or libraries that could be replaced with smaller alternatives.

## Tool 1: webpack-bundle-analyzer

The most widely used tool for Webpack projects. Generates an interactive treemap showing bundle contents by size.

### Setup

```bash
npm install --save-dev webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',      // Generates report.html (CI-friendly)
      // analyzerMode: 'server',   // Opens interactive browser view
      openAnalyzer: false,
      reportFilename: 'bundle-report.html',
    }),
  ],
};

// Or as one-off via CLI
ANALYZE=true npx webpack

// package.json script
"analyze": "ANALYZE=true npm run build"
```

### Reading the Treemap

```
webpack-bundle-analyzer output:

┌─────────────────────────────────────────────────────────────────┐
│  main.js (2.1MB gzip: 680KB)                                   │
│                                                                 │
│  ┌───────────────────────┐  ┌──────────────────────────────┐   │
│  │  node_modules (1.8MB) │  │  src (300KB)                 │   │
│  │                       │  │                              │   │
│  │  ┌──────────────────┐ │  │  ┌────────┐  ┌───────────┐  │   │
│  │  │  moment (230KB)  │ │  │  │ pages  │  │ components│  │   │
│  │  └──────────────────┘ │  │  └────────┘  └───────────┘  │   │
│  │  ┌────────────────┐   │  └──────────────────────────────┘   │
│  │  │  lodash (72KB) │   │                                     │
│  │  └────────────────┘   │                                     │
│  │  ┌────────────────┐   │                                     │
│  │  │ @mui (450KB)   │   │                                     │
│  │  └────────────────┘   │                                     │
│  └───────────────────────┘                                     │
└─────────────────────────────────────────────────────────────────┘

What to look for:
  - Large rectangles in node_modules (obvious targets)
  - Libraries you don't recognize (transitive dependencies)
  - The same module appearing twice (duplication)
  - moment.js (almost always replaceable with date-fns)
```

## Tool 2: Vite Bundle Visualizer (rollup-plugin-visualizer)

For Vite and Rollup projects:

```bash
npm install --save-dev rollup-plugin-visualizer
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      open: true,            // Opens browser after build
      gzipSize: true,        // Show gzip sizes (more realistic)
      brotliSize: true,      // Show brotli sizes
      filename: 'dist/stats.html',
      template: 'treemap',   // 'treemap' | 'sunburst' | 'network'
    }),
  ],
});
```

```bash
npm run build
# Opens stats.html automatically
```

## Tool 3: source-map-explorer

Source map explorer works with any bundler — it uses source maps to attribute each byte to its original source file. Unlike treemap tools, it shows the **actual post-tree-shaking bundle content**:

```bash
npm install --save-dev source-map-explorer

# Requires source maps to be generated
# Build with source maps:
# Webpack: devtool: 'source-map'
# Vite: build.sourcemap: true

# Analyze
npx source-map-explorer dist/main.js --html output.html

# Or multiple chunks
npx source-map-explorer 'dist/*.js' --html output.html
```

```
source-map-explorer output (text format):

File: dist/main.js
Bytes    File
230,412  node_modules/moment/moment.js          ← 23% of bundle
 71,204  node_modules/lodash/lodash.js          ← 7%
 45,123  node_modules/@mui/material/...
 23,811  src/pages/Dashboard.tsx
 18,432  src/components/DataTable.tsx
...

Total: 1,012,345 bytes (982KB)
```

The key difference from webpack-bundle-analyzer: source-map-explorer shows the **real** contents of the final bundle, accounting for tree shaking. If a library is 230KB in source-map-explorer, that 230KB is actually in the bundle.

## Tool 4: Next.js Bundle Analyzer

```bash
npm install --save-dev @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // your next.js config
});
```

```bash
ANALYZE=true npm run build
# Opens client-side and server-side bundle reports
```

## Interpreting Results: What to Look For

### 1. Large Third-Party Libraries

```
Red flags (common large libraries with lighter alternatives):

moment (230KB)          → date-fns (tree-shakeable, ~5KB per function)
                        → dayjs (2KB core)
lodash (72KB full)      → lodash-es (tree-shakeable) or native JS
                        → just-* family of micro-utilities
axios (14KB)            → native fetch (0KB, built-in)
highlight.js (full)     → import only needed languages
react-icons (all)       → import individual icons

Check: npm.anvaka.com — visual dependency tree
Check: bundlephobia.com — cost of any npm package
```

### 2. Duplicate Dependencies

```
Duplication happens when:
  - Multiple packages depend on different versions of the same library
  - yarn/npm can't hoist them due to semver incompatibility

Signs in bundle analyzer:
  - React appearing twice (react@17 + react@18 in same bundle — fatal)
  - lodash appearing 3 times (different parts of the tree pinned to different versions)
  - moment locale data included multiple times

Diagnosis:
```

```bash
# Find duplicates with npm
npm ls react

# Find duplicates with yarn (deduplicate)
yarn dedupe

# Check with duplicate-package-checker-webpack-plugin
npm install --save-dev duplicate-package-checker-webpack-plugin
```

```javascript
// webpack.config.js
const DuplicatePackageCheckerPlugin = require('duplicate-package-checker-webpack-plugin');

module.exports = {
  plugins: [
    new DuplicatePackageCheckerPlugin({
      verbose: true,
      emitError: true, // Fail build on duplicates
    }),
  ],
};
```

### 3. Moment.js Locale Data

Moment.js is a classic — it bundles ALL locale files by default (~160KB of locale data):

```javascript
// webpack.config.js — strip moment locales
const webpack = require('webpack');

module.exports = {
  plugins: [
    // Only include English locale (saves ~160KB)
    new webpack.IgnorePlugin({
      resourceRegExp: /^\.\/locale$/,
      contextRegExp: /moment$/,
    }),
  ],
};

// Then import locales manually if needed:
import 'moment/locale/fr';
import 'moment/locale/de';

// Better: migrate to date-fns which has no locale bundling problem
import { formatDistanceToNow } from 'date-fns/formatDistanceToNow';
import { fr } from 'date-fns/locale/fr';
```

### 4. Importing Entire Libraries for One Function

```typescript
// ❌ Imports entire lodash (72KB minified)
import _ from 'lodash';
const result = _.debounce(fn, 300);

// ✅ Import from lodash-es (tree-shakeable)
import { debounce } from 'lodash-es';

// ✅ Or use native equivalent (0KB)
function debounce<T extends (...args: any[]) => any>(fn: T, delay: number): T {
  let timer: ReturnType<typeof setTimeout>;
  return ((...args: any[]) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  }) as T;
}

// ❌ Imports all icons (can be MBs)
import { FiHome, FiUser } from 'react-icons/fi';
// react-icons/fi barrel re-exports ALL feather icons

// ✅ Import directly from file
import FiHome from 'react-icons/fi/FiHome';
```

### 5. All-or-Nothing UI Libraries

```typescript
// ❌ Import entire MUI (huge)
import Button from '@mui/material/Button';
import TextField from '@mui/material/TextField';
// Even named imports from @mui/material may not tree-shake in v4

// ✅ MUI v5+ with ESM imports tree-shakes correctly
import { Button } from '@mui/material';
import { TextField } from '@mui/material';
// Check: MUI v5 declares "sideEffects": false — works with modern bundlers

// ✅ Consider lighter alternatives for bundle-sensitive apps
// Radix UI (~few KB per component)
// Headless UI (~5KB)
// shadcn/ui (copies components into your codebase — zero bundle overhead)
```

## Setting Up Automated Bundle Budget Checks

```javascript
// webpack.config.js — fail build if bundle exceeds budget
module.exports = {
  performance: {
    maxAssetSize: 250_000,         // 250KB per asset
    maxEntrypointSize: 500_000,    // 500KB entry point
    hints: 'error',                // 'warning' in dev, 'error' in CI
  },
};
```

```javascript
// Custom bundle size check script
// scripts/check-bundle-size.js
const { execSync } = require('child_process');
const path = require('path');
const fs = require('fs');

const LIMITS = {
  'dist/main.js': 300_000,      // 300KB max for main chunk
  'dist/vendor.js': 500_000,    // 500KB max for vendor chunk
};

for (const [file, limit] of Object.entries(LIMITS)) {
  const filePath = path.resolve(file);
  const stat = fs.statSync(filePath);
  if (stat.size > limit) {
    console.error(
      `Bundle size exceeded: ${file} is ${stat.size} bytes (limit: ${limit})`
    );
    process.exit(1);
  }
}
console.log('Bundle size check passed');
```

```yaml
# GitHub Actions — bundle size check in CI
- name: Build
  run: npm run build

- name: Analyze bundle
  run: npx source-map-explorer 'dist/*.js' --json > bundle-stats.json

- name: Check bundle size
  run: node scripts/check-bundle-size.js
```

## bundlesize / size-limit Packages

```bash
# size-limit — most popular bundle size CI tool
npm install --save-dev @size-limit/preset-app size-limit
```

```json
// package.json
{
  "size-limit": [
    {
      "path": "dist/main.js",
      "limit": "200 KB",         // gzipped limit
      "gzip": true
    },
    {
      "path": "dist/vendor.js",
      "limit": "300 KB",
      "gzip": true
    }
  ],
  "scripts": {
    "size": "size-limit",
    "analyze": "size-limit --why"
  }
}
```

```bash
npm run size
# > @myapp/app@1.0.0 size
# > Package size limit has exceeded the limit
# ✗ dist/main.js
#   Size: 210 KB with all dependencies, minified and gzipped
#   Limit: 200 KB

npm run analyze   # Shows why the limit is exceeded (uses webpack internally)
```

## Practical Analysis Workflow

```
Bundle analysis workflow:

1. Baseline measurement
   → Run: npm run build && npx source-map-explorer 'dist/*.js'
   → Record total size, top 5 largest modules

2. Identify quick wins
   → Any library > 50KB? Check bundlephobia.com for alternatives
   → Any duplicates? Fix with npm dedupe / webpack aliases
   → Moment.js? Replace with date-fns

3. Verify tree shaking
   → Build with sourcemap + no minify
   → grep for "unused harmony export"
   → Check sideEffects in offending package.json

4. Code split large routes
   → Identify routes > 50KB
   → Add React.lazy / dynamic imports
   → Measure per-route chunk sizes

5. Establish budget
   → Set size-limit thresholds in CI
   → Track deltas on each PR
```

## Common Mistakes

### 1. Analyzing Without Source Maps

```bash
# ❌ source-map-explorer without source maps
# Shows only minified module names — unreadable

# ✅ Build with source maps first
# vite.config.ts: build.sourcemap: true
# webpack.config.js: devtool: 'source-map'
npm run build && npx source-map-explorer dist/main.js
```

### 2. Only Analyzing Production, Not Per-Route

```typescript
// ❌ Only checking total bundle size
// Total: 400KB — "fine"

// ✅ Check per-route initial load
// Route /dashboard loads: 400KB — that's the whole app on first visit
// 300KB of that is used only in /admin which requires authentication

// Code split by route:
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Admin = React.lazy(() => import('./pages/Admin'));
// Now /dashboard initial load: ~100KB
```

### 3. Ignoring the Vendor Chunk

```javascript
// ❌ Everything in one bundle — no caching benefit
// main.js: 900KB (app code + all dependencies)
// When you deploy a new version, users re-download 900KB

// ✅ Separate vendor chunk — cached across deploys
optimization: {
  splitChunks: {
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        chunks: 'all',
      },
    },
  },
},
// vendors.abc123.js: 600KB (changes only when deps change — cached)
// main.def456.js: 300KB (changes on every deploy — small re-download)
```

## Interview Questions

### 1. What is the difference between webpack-bundle-analyzer and source-map-explorer?

**Answer:** webpack-bundle-analyzer generates a treemap from Webpack's internal module stats and shows modules as they exist before the final bundle is written — it may show code that tree shaking will ultimately remove. source-map-explorer reads the actual output bundle and uses source maps to attribute each byte back to its original source. It shows what's truly in the final bundle after all optimizations, making it more accurate for measuring real bundle cost. For Webpack projects use both: webpack-bundle-analyzer for understanding the module structure and tree-shaking gaps, source-map-explorer for the ground truth of what users download.

### 2. How would you investigate and fix a bundle size regression in CI?

**Answer:** Set up `size-limit` or a custom script that compares bundle sizes between the PR branch and main. When a regression is detected: (1) run `source-map-explorer` on both branches and compare the top modules; (2) identify the new or grown module; (3) check if it's a direct or transitive dependency using `npm ls <package-name>`; (4) check bundlephobia.com for the package's size; (5) evaluate alternatives (smaller library, tree-shakeable version, lazy loading). For a new dependency, require the PR author to justify the size cost. Automate this as a required CI check so bundles never regrow silently.

### 3. Why does moment.js appear so large in bundle analysis?

**Answer:** Moment.js bundles all locale data by default — roughly 160KB of translation strings for dozens of locales that most apps will never use. Webpack's `IgnorePlugin` can strip locale files, but this requires manual opt-in of needed locales. The deeper problem is that moment is not tree-shakeable (CommonJS) and its API is designed around mutable instances which makes partial inclusion impossible. The standard recommendation is to migrate to `date-fns` (tree-shakeable, named exports, ~5KB per used function) or `dayjs` (2KB core, plugin-based).

### 4. What is duplicate dependency bundling and how do you fix it?

**Answer:** Duplicate bundling occurs when the same package appears multiple times in the bundle — typically because different versions exist in the `node_modules` tree that npm/yarn can't deduplicate due to semver incompatibility. The most dangerous case is React duplicated (React 17 and React 18 in the same bundle — hooks break). Detection: `npm ls react` or `duplicate-package-checker-webpack-plugin`. Fixes: (1) `npm dedupe` or `yarn dedupe` — resolves compatible duplicates; (2) webpack `resolve.alias` to force a single version: `{ react: path.resolve('./node_modules/react') }`; (3) update the conflicting package to use the same major version.

### 5. What metrics should you track in a bundle size monitoring system?

**Answer:** Track: total initial bundle size (gzipped), per-route initial load size, largest individual chunk size, and count of chunks loaded on initial navigation. Monitor trends over time (not just absolute thresholds) — a 2KB growth per PR compounds to hundreds of KB over months. Set PR-level budgets with size-limit so no single PR can exceed a threshold. For a large app, also track "bundle cost per feature" — when a feature flag gates large code paths, measure the cost of each path. Segment metrics by entry point (main app, admin panel) since they have different performance budgets.
