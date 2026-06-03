# Webpack 5

## What It Does

Webpack bundles JavaScript modules (and other assets) into optimized files for the browser. It analyzes your import graph, applies transformations (loaders), and outputs bundles.

---

## Core Concepts

```js
// webpack.config.js
module.exports = {
  // Entry point(s) — where Webpack starts the dependency graph
  entry: './src/index.tsx',

  // Output — where to emit bundles
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js', // content hash for cache busting
    clean: true, // clear dist/ before each build
  },

  // Resolve — how to find modules
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
    alias: { '@': path.resolve(__dirname, 'src') },
  },

  // Loaders — transform non-JS files
  module: {
    rules: [
      {
        test: /\.(ts|tsx)$/,
        use: 'ts-loader', // or 'babel-loader' with @babel/preset-typescript
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader'],
      },
      {
        test: /\.(png|jpg|gif|svg)$/,
        type: 'asset/resource', // Asset Modules (Webpack 5, replaces file-loader)
      },
    ],
  },

  // Plugins — extend Webpack capabilities
  plugins: [
    new HtmlWebpackPlugin({ template: './src/index.html' }),
    new MiniCssExtractPlugin({ filename: '[name].[contenthash].css' }),
  ],
};
```

---

## Code Splitting

### Dynamic Imports (Automatic Split Point)

```ts
// Webpack creates a separate chunk for this import
const LazyComponent = React.lazy(() => import('./HeavyComponent'));

// Or in route config:
const routes = [
  {
    path: '/admin',
    component: React.lazy(() => import('./admin/AdminDashboard')),
  }
];
```

### `splitChunks` (Vendor Chunk)

```js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        chunks: 'all',
      },
      react: {
        test: /[\\/]node_modules[\\/](react|react-dom|react-router)[\\/]/,
        name: 'react-vendor',
        chunks: 'all',
        priority: 10, // higher priority = checked first
      },
    },
  },
},
```

Result: React/React DOM is in a separate `react-vendor.js` chunk that can be cached independently from your app code.

---

## Module Federation (Micro-Frontends)

Module Federation lets multiple Webpack builds share code at runtime:

```js
// Host app webpack.config.js
new ModuleFederationPlugin({
  name: 'host',
  remotes: {
    checkout: 'checkout@https://checkout.example.com/remoteEntry.js',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
})

// Remote (checkout) webpack.config.js
new ModuleFederationPlugin({
  name: 'checkout',
  filename: 'remoteEntry.js',
  exposes: { './App': './src/CheckoutApp' },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
})

// Host loads remote dynamically
const CheckoutApp = React.lazy(() => import('checkout/App'));
```

---

## Persistent Caching

```js
// Webpack 5 filesystem cache — dramatically speeds up rebuilds
cache: {
  type: 'filesystem',
  buildDependencies: {
    config: [__filename], // invalidate cache when config changes
  },
},
```

After first build, subsequent builds only recompile changed modules. Typical 2-3x speedup for incremental builds.

---

## Tree Shaking

Tree shaking eliminates unused exports. Prerequisites:
- ES modules (`import`/`export`, not CommonJS `require`)
- `sideEffects: false` in `package.json` (or list files with side effects)
- `mode: 'production'` (enables Terser for dead code elimination)

```json
// package.json
{
  "sideEffects": ["*.css", "./src/polyfills.ts"]  // these files have side effects
}
```

---

## Asset Modules (Webpack 5)

Replaces `file-loader`, `url-loader`, `raw-loader`:

```js
// type: 'asset/resource' → emits file, exports URL
{ test: /\.(png|jpg|gif)$/, type: 'asset/resource' }

// type: 'asset/inline' → base64-encodes into bundle
{ test: /\.svg$/, type: 'asset/inline' }

// type: 'asset' → auto decide (inline if < 8KB, resource if larger)
{ test: /\.(png|jpg)$/, type: 'asset', parser: { dataUrlCondition: { maxSize: 8 * 1024 } } }

// type: 'asset/source' → exports raw string (like raw-loader)
{ test: /\.txt$/, type: 'asset/source' }
```

---

## Common Interview Questions

**Q: What's the difference between loaders and plugins?**
Loaders transform individual files (TypeScript → JS, Sass → CSS). They run during the module resolution phase. Plugins are broader — they hook into Webpack's lifecycle events to perform optimizations, asset management, and more (HtmlWebpackPlugin, MiniCssExtractPlugin).

**Q: How does `contenthash` help with caching?**
`[contenthash]` in the filename changes only when the file's content changes. Vendor bundles (React, etc.) get a stable hash that browsers can cache long-term. App code hash changes on each deployment, forcing re-download.

**Q: What's the difference between Webpack 5 and Vite?**
Webpack bundles everything (slow cold start). Vite serves unbundled ESM in dev (instant cold start), only transforms on demand. Vite's build uses Rollup. Vite is now preferred for new projects; Webpack for enterprise apps with Module Federation needs.
