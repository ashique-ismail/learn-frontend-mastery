# Bundlers: Vite, Webpack, esbuild, Rollup, and Turbopack

## Overview

A JavaScript bundler takes a graph of modules — JavaScript, CSS, images, fonts, and other assets — and produces one or more optimized output files for deployment. Bundlers handle module resolution, code transformation, tree shaking, code splitting, asset hashing, and a dozen other concerns that make modern web applications work at scale.

The bundler landscape has shifted dramatically since Webpack dominated the 2015-2020 era. esbuild's introduction of Go-based native bundling (10-100x faster than JavaScript-based tools) forced a rethink of what "fast" means in frontend tooling. Vite built a developer experience layer on top of esbuild for dev and Rollup for production. Turbopack is Webpack's Rust-based successor. Understanding each tool's architecture, tradeoffs, and appropriate use cases is expected knowledge for senior engineers owning frontend infrastructure.

## How Bundlers Work: Core Phases

Every bundler, regardless of implementation, performs the same conceptual phases:

```
1. Entry point(s)
2. Module resolution    — resolve() turns specifiers to file paths
3. Loading             — load() reads file contents
4. Transformation      — transform() runs per-file (TS→JS, JSX→JS, CSS→JS)
5. Dependency graph    — parse imports, build a DAG of all modules
6. Optimization        — tree shake, scope hoist, minify
7. Code splitting      — determine chunk boundaries
8. Output             — write chunks with content hashes to disk
```

## Webpack

### Architecture

Webpack is a module bundler with a plugin-based architecture. It uses a concept of **loaders** (transform individual files) and **plugins** (operate on the full compilation). Every asset type (CSS, images, fonts, WASM) is treated as a module in the dependency graph.

```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  entry: {
    main: './src/index.js',
    admin: './src/admin.js'
  },
  output: {
    filename: '[name].[contenthash].js',  // content hashing for caching
    path: path.resolve(__dirname, 'dist'),
    clean: true
  },
  mode: 'production',
  module: {
    rules: [
      {
        test: /\.(ts|tsx)$/,
        use: 'ts-loader',
        exclude: /node_modules/
      },
      {
        test: /\.css$/,
        use: [MiniCssExtractPlugin.loader, 'css-loader', 'postcss-loader']
      },
      {
        test: /\.(png|svg|jpg)$/,
        type: 'asset/resource'
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({ template: './src/index.html' }),
    new MiniCssExtractPlugin({ filename: '[name].[contenthash].css' })
  ],
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    },
    runtimeChunk: 'single'
  }
};
```

### Webpack Dev Server and HMR

Webpack's development server provides Hot Module Replacement (HMR): changed modules are re-evaluated in the browser without a full page reload, preserving application state.

```javascript
// webpack.config.js
module.exports = {
  devServer: {
    port: 3000,
    hot: true,                    // enable HMR
    historyApiFallback: true,     // SPA routing
    proxy: {
      '/api': 'http://localhost:8080' // dev proxy
    }
  }
};
```

HMR works by injecting a WebSocket connection into the bundle. When a file changes, Webpack recompiles only the affected module subgraph and pushes an update over the WebSocket. The runtime applies the update by re-requiring changed modules and propagating the update boundary.

### Webpack 5 Key Features

```javascript
// Module Federation — share modules across independently deployed apps
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app2: 'app2@http://localhost:3002/remoteEntry.js'
      },
      shared: ['react', 'react-dom']
    })
  ]
};
```

```javascript
// Asset modules — replaces file-loader, url-loader
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|jpg|gif)$/i,
        type: 'asset',              // auto: inline if <8kb, else resource
        parser: { dataUrlCondition: { maxSize: 8 * 1024 } }
      }
    ]
  }
};
```

### Webpack Strengths and Weaknesses

Strengths: mature ecosystem, extensive loader/plugin library, Module Federation for micro-frontends, extremely configurable.

Weaknesses: slow startup in large projects (full rebuild on first run), configuration complexity, poor defaults requiring significant boilerplate.

## esbuild

### Architecture

esbuild is written in Go and builds a highly parallelized native bundler. It is typically 10-100x faster than JavaScript-based bundlers. It handles TypeScript, JSX, ESM, CJS, tree shaking, code splitting, and minification natively without plugins for common transforms.

```bash
# CLI
esbuild src/index.ts \
  --bundle \
  --outfile=dist/bundle.js \
  --platform=browser \
  --target=es2020 \
  --minify \
  --sourcemap \
  --splitting \
  --format=esm
```

```javascript
// JavaScript API
const esbuild = require('esbuild');

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  outdir: 'dist',
  format: 'esm',
  splitting: true,       // code splitting (requires format: 'esm')
  target: ['es2020', 'chrome90', 'firefox88'],
  minify: true,
  sourcemap: true,
  external: ['react', 'react-dom'],  // exclude from bundle
  define: {
    'process.env.NODE_ENV': '"production"'
  },
  plugins: [
    // plugins are minimal — most transforms are built in
  ]
});
```

### What esbuild Does Not Do

esbuild intentionally omits features to stay fast:
- No TypeScript type checking (transpile-only, like `tsc --noEmit` + esbuild)
- Limited plugin API compared to Webpack/Rollup
- No `require()` of ESM (no CJS-to-ESM full interop)
- No first-class Module Federation
- CSS handling is basic (no CSS Modules natively)

```bash
# esbuild only transpiles TypeScript — no type checking
# Run tsc separately for type errors
npx tsc --noEmit           # type check only
npx esbuild src/*.ts --bundle --outdir=dist  # transpile only
```

### esbuild as a Library Component

esbuild is commonly used as the transform layer inside other tools:
- Vite uses esbuild for dev server transforms and dependency pre-bundling
- Jest uses `esbuild-jest` or `@swc/jest` for test transforms
- `tsup` uses esbuild for TypeScript library builds
- Nx uses esbuild as an executor

## Rollup

### Architecture

Rollup pioneered tree shaking and is the reference implementation for ESM bundling. It is designed primarily for libraries (producing clean, flat ESM/CJS output) rather than applications. Its output is notably cleaner than Webpack's — no module runtime wrapper, just the code itself.

```javascript
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import typescript from '@rollup/plugin-typescript';
import { terser } from 'rollup-plugin-terser';

export default {
  input: 'src/index.ts',
  external: ['react', 'react-dom'], // peer deps — don't bundle
  output: [
    {
      file: 'dist/index.cjs',
      format: 'cjs',
      sourcemap: true
    },
    {
      file: 'dist/index.mjs',
      format: 'esm',
      sourcemap: true
    }
  ],
  plugins: [
    resolve(),
    commonjs(),
    typescript({ tsconfig: './tsconfig.build.json' }),
    terser()
  ],
  treeshake: {
    preset: 'recommended',
    moduleSideEffects: false
  }
};
```

### Scope Hoisting

Rollup's output is significantly smaller than Webpack's for libraries because it uses scope hoisting: all modules are merged into a single scope, eliminating module wrapper functions.

```javascript
// Webpack output (simplified) — wrapped modules
/*** ./src/add.js ***/
((__unused_webpack_module, exports) => {
  exports.add = function add(a, b) { return a + b; };
})

// Rollup output — flat scope hoisting
function add(a, b) { return a + b; }
// No wrapper, no module runtime overhead
```

### Rollup for Libraries

Rollup is the standard choice for publishing npm packages:

```javascript
// rollup.config.js for a dual-format library
export default {
  input: 'src/index.ts',
  output: [
    { format: 'esm',  file: 'dist/index.mjs', sourcemap: true },
    { format: 'cjs',  file: 'dist/index.cjs', sourcemap: true },
    { format: 'umd',  file: 'dist/index.umd.js', name: 'MyLib', globals: { react: 'React' } }
  ]
};
```

## Vite

### Architecture

Vite uses a two-phase approach:
- **Development**: Native browser ESM — no bundling, serve files directly transformed by esbuild. The browser's module system handles resolution. Only changed files are transformed.
- **Production**: Rollup bundling — full optimization, tree shaking, code splitting, asset hashing.

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  
  build: {
    target: 'es2020',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom']
        }
      }
    },
    cssCodeSplit: true,
    assetsInlineLimit: 4096  // inline assets <4KB as base64
  },
  
  server: {
    port: 3000,
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true }
    }
  },
  
  optimizeDeps: {
    include: ['lodash-es'],  // pre-bundle specific CJS deps
    exclude: ['@company/ui'] // don't pre-bundle (it's ESM)
  }
});
```

### Dependency Pre-bundling

One of Vite's most important internals: on first server start, Vite uses esbuild to convert CJS dependencies to ESM and consolidate "deep-imports" packages (like lodash-es with hundreds of files) into a single ESM file. This prevents the browser from issuing hundreds of HTTP requests for a single `import { debounce } from 'lodash-es'`.

```bash
# Pre-bundled deps are cached in node_modules/.vite/deps/
# Invalidated when package.json or vite.config.ts changes

# Force re-prebundle
npx vite --force
```

### Vite Plugin API

Vite extends Rollup's plugin API with dev-server-specific hooks:

```typescript
// vite plugin
import type { Plugin } from 'vite';

function myPlugin(): Plugin {
  return {
    name: 'my-plugin',
    
    // Rollup hooks — used in both dev and build
    resolveId(id) { /* custom resolution */ },
    load(id) { /* custom loading */ },
    transform(code, id) { /* per-file transformation */ },
    
    // Vite-specific hooks
    configureServer(server) {
      server.middlewares.use('/custom', (req, res) => {
        res.end('custom response');
      });
    },
    
    handleHotUpdate({ file, server }) {
      if (file.endsWith('.special')) {
        server.ws.send({ type: 'full-reload' });
        return [];
      }
    }
  };
}
```

### HMR in Vite

Vite's HMR is faster than Webpack's because there is no rebuild phase. Vite transforms only the single changed file and sends a hot update over WebSocket. The browser fetches only the changed module.

```javascript
// Accept HMR updates in your module
if (import.meta.hot) {
  import.meta.hot.accept('./child.js', (newChild) => {
    // Apply update from newChild
  });
  
  import.meta.hot.dispose(() => {
    // Clean up before replacement
  });
}
```

## Turbopack

### Architecture

Turbopack is Vercel's Rust-based bundler, positioned as Webpack's successor. It is built into Next.js 13+ (opt-in) and aims for sub-second startup and near-instant HMR at any scale.

Key architectural innovation: **incremental computation with a demand-driven call graph**. Turbopack computes only what is needed for the current request and caches every computation granularly. A change to one file invalidates only the functions that transitively depend on that file's output, not the entire compilation.

```bash
# Enable in Next.js
next dev --turbo

# next.config.js
module.exports = {
  experimental: {
    turbo: {
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js'
        }
      }
    }
  }
};
```

Turbopack is still maturing (production builds stabilized in Next.js 15). It is not yet a standalone tool with the configuration surface of Webpack.

## Comparison Table

| Feature | Webpack 5 | esbuild | Rollup | Vite | Turbopack |
|---|---|---|---|---|---|
| Language | JavaScript | Go | JavaScript | JS + esbuild + Rollup | Rust |
| Primary use | Apps | Apps + Libraries | Libraries | Apps | Apps (Next.js) |
| Dev speed | Medium | Very fast | Fast | Very fast | Extremely fast |
| Build speed | Medium | Very fast | Fast | Fast | Fast |
| Tree shaking | Good | Good | Excellent | Excellent (Rollup) | Good |
| Code splitting | Excellent | Good | Good | Excellent (Rollup) | Good |
| Plugin ecosystem | Huge | Small | Large | Large (extends Rollup) | Early |
| HMR | Yes | No (via tools) | No (via tools) | Yes (native ESM) | Yes |
| Module Federation | Yes | No | No | Partial | Planned |
| CSS handling | Via loaders | Basic | Via plugins | Native | Via Next.js |
| CJS interop | Excellent | Good | Via plugin | Via pre-bundling | Good |
| Config complexity | High | Low | Medium | Low | Low (Next.js only) |
| TypeScript support | Via loader | Transpile-only | Via plugin | Via esbuild | Native |

## Best Practices

### 1. Match Tool to Use Case

```
- App with framework: Vite (React/Vue/Svelte) or Next.js (Turbopack)
- Library for npm: Rollup or tsup (esbuild-based)
- Complex enterprise app with Module Federation: Webpack 5
- Build tool integration / fast transforms: esbuild API
```

### 2. Always Use Content Hashing in Production

```javascript
// Webpack
output: { filename: '[name].[contenthash:8].js' }

// Vite (default in production)
build: { rollupOptions: { output: { entryFileNames: '[name]-[hash].js' } } }
```

Content hashes ensure browsers use cached bundles for unchanged files and bust cache only for changed ones.

### 3. Separate Vendor Chunk

```javascript
// Vite / Rollup
manualChunks: {
  vendor: ['react', 'react-dom', 'react-router-dom']
}

// Webpack
splitChunks: {
  cacheGroups: {
    vendor: { test: /node_modules/, name: 'vendors', chunks: 'all' }
  }
}
```

Libraries change less frequently than app code. A separate vendor chunk stays cached across deployments.

### 4. Analyze Bundles Before Each Major Release

```bash
# Vite
npx vite-bundle-visualizer

# Webpack
npx webpack-bundle-analyzer dist/stats.json

# General
npx source-map-explorer dist/*.js
```

### 5. Use externals for Libraries

```javascript
// Do not bundle peer dependencies into your library
// Rollup
external: ['react', 'react-dom', /^@radix-ui\//]

// Webpack
externals: { react: 'React', 'react-dom': 'ReactDOM' }
```

## Interview Questions

### Q1: Why is Vite's dev server faster than Webpack's for large apps?

**Answer:** Webpack bundles the entire application on startup — every file is transformed, dependency-graphed, and emitted before the dev server starts serving. For large apps, this can take 30-60+ seconds. Vite serves files unbundled via native browser ESM. On startup, Vite only pre-bundles CJS dependencies using esbuild (extremely fast) and starts serving immediately. The browser requests only the modules it needs to render the current page. When a file changes, Vite transforms and serves only that single file. This means startup time scales with the number of files on the critical path for the current page, not the total number of files in the project.

### Q2: What is Module Federation and when would you use it?

**Answer:** Module Federation (Webpack 5+) allows independently deployed JavaScript applications to share modules at runtime. Application A can expose a component or utility, and Application B fetches and uses it directly from A's deployed URL — without A's code being in B's bundle at build time. This enables micro-frontend architectures where multiple teams deploy independently but share UI components or state. Use cases include large enterprise products split across teams, gradually migrating a monolith, or implementing shell/remote architectures. The key benefit is true runtime code sharing; the tradeoff is complexity in versioning, dependency alignment, and failure isolation.

### Q3: Why is Rollup preferred for library builds while Webpack is preferred for application builds?

**Answer:** Rollup's scope hoisting produces minimal output: modules are merged into a single flat scope without wrapper functions, making it ideal for libraries where bundle size directly affects consumers. Rollup outputs clean ESM that is itself tree-shakeable by consumers' bundlers. Webpack excels at application concerns: HMR during development, complex code splitting strategies, processing non-JavaScript assets (CSS, images, fonts) as first-class modules, Module Federation for micro-frontends, and a massive ecosystem of loaders for every asset type. Libraries rarely need these features; applications rely on them heavily.

### Q4: How does esbuild achieve its build speed advantage?

**Answer:** esbuild's performance comes from several architectural decisions: it is written in Go (compiled to native machine code, vs. JavaScript's JIT), it parallelizes work across all available CPU cores from the start, it uses a single pass through the dependency graph rather than multiple phases, it avoids expensive JavaScript-to-JavaScript transform pipelines (handles TypeScript, JSX, and minification natively), and it minimizes heap allocations. The 10-100x speedup over Webpack is primarily explained by the native execution model and parallelism. JavaScript bundlers spend significant time in V8's JIT and garbage collector; esbuild avoids both.

### Q5: What is Vite's dependency pre-bundling and why is it necessary?

**Answer:** Vite's dev server serves files as native ESM. Two problems arise with raw npm packages: first, many packages are still CJS and cannot be served as ESM natively. Second, packages like lodash-es consist of hundreds of individual ESM files — serving them unbundled would cause the browser to make hundreds of HTTP requests for a single import. Pre-bundling uses esbuild to convert CJS packages to ESM and consolidate multi-file ESM packages into a single file. The pre-bundled packages are cached in `node_modules/.vite/deps/` and served with long max-age headers. This happens once on first run and is invalidated only when dependencies change.

## Common Pitfalls

### 1. Forgetting mode: 'production' in Webpack

```javascript
// Without this, tree shaking, minification, and optimizations are disabled
module.exports = {
  mode: 'production' // required for tree shaking to work
};
```

### 2. Over-splitting into Too Many Chunks

Every chunk adds an HTTP round trip. Splitting 500 components into 500 chunks is counterproductive — HTTP/2 helps but doesn't eliminate overhead. Benchmark before over-optimizing.

### 3. Including Node.js Built-ins in Browser Bundles

```javascript
// Node.js built-ins are not available in browsers
// Webpack will try to polyfill them (v4 default) or fail (v5 default)
import { readFileSync } from 'fs'; // ERROR in browser bundle

// Set target to browser explicitly
module.exports = {
  target: 'web' // disables Node.js built-in polyfills
};
```

### 4. Not Externalizing Peer Dependencies in Libraries

```javascript
// If you bundle react into your library:
// - Consumer gets two copies of React → hook rule violations
// - Library bundle is unnecessarily large

// Rollup: always externalize peer deps
external: ['react', 'react-dom']
```

### 5. Relying on Vite Dev Behavior in Production

Vite dev server (unbundled ESM) behaves differently from Rollup production builds. Always test against `vite build && vite preview`, not just `vite dev`.

## Resources

### Official Documentation
- [Webpack Documentation](https://webpack.js.org/)
- [esbuild Documentation](https://esbuild.github.io/)
- [Rollup Documentation](https://rollupjs.org/)
- [Vite Documentation](https://vitejs.dev/)
- [Turbopack Documentation](https://turbo.build/pack)

### Articles and Guides
- [Bundler benchmarks](https://github.com/evanw/bundler-benchmark)
- [Vite: Why Not Bundle During Dev?](https://vitejs.dev/guide/why)
- [Module Federation Documentation](https://module-federation.io/)

### Tools
- [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
- [rollup-plugin-visualizer](https://github.com/btd/rollup-plugin-visualizer)
- [tsup](https://tsup.egoist.dev/): esbuild-based library bundler with zero config
- [unbuild](https://github.com/unjs/unbuild): Rollup-based with stub mode for dev
