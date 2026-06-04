# Rspack

## What It Is

Rspack is a Rust-based JavaScript bundler built by ByteDance that aims to be API-compatible with Webpack 5 while being dramatically faster. It can be used as a drop-in replacement for Webpack in many projects.

---

## Performance Claims

- **10-20x faster** cold builds than Webpack
- **3-5x faster** rebuild/HMR than Webpack
- Built with Rust's parallelism — module transformations run on multiple threads

---

## Webpack Compatibility

Rspack implements most of Webpack's configuration surface:

```js
// webpack.config.js vs rspack.config.js — nearly identical
module.exports = {
  entry: './src/index.tsx',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
  },
  module: {
    rules: [
      {
        test: /\.(tsx?|jsx?)$/,
        // Built-in TypeScript and React support — no ts-loader or babel-loader needed
        use: [{ loader: 'builtin:swc-loader', options: { jsc: { parser: { syntax: 'typescript', tsx: true } } } }],
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  plugins: [
    new HtmlRspackPlugin({ template: './src/index.html' }), // native Rspack plugin
  ],
};
```

---

## Built-in Features (No Loader Required)

```js
// rspack.config.js
module.exports = {
  builtins: {
    html: [{ template: './index.html' }],   // replaces HtmlWebpackPlugin
    css: { modules: true },                  // CSS Modules
    postcss: { pxtorem: {} },               // PostCSS
    minifyOptions: { passes: 1 },           // built-in minification
  },
};
```

Rspack has native implementations for:
- **TypeScript** (via SWC, built-in)
- **JSX/TSX** transformation
- **CSS Modules**
- **HTML generation** (HtmlRspackPlugin)
- **Asset processing** (same Asset Modules API as Webpack 5)
- **Tree shaking**

---

## Module Federation

```js
const { ModuleFederationPlugin } = require('@module-federation/enhanced/rspack');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        mfe: 'mfe@https://mfe.example.com/remoteEntry.js',
      },
      shared: { react: { singleton: true } },
    }),
  ],
};
```

---

## Migration Strategy

### From Webpack

```bash
npm install -D @rspack/cli @rspack/core

# Replace webpack.config.js with rspack.config.js
# Most configuration is identical
```

Common incompatibilities:
- Some Webpack plugins have no Rspack equivalent yet
- Custom loaders that rely on Webpack internals may not work
- Babel-specific plugins may need SWC equivalents

```js
// Replace babel-loader with builtin:swc-loader
// webpack.config.js
{ loader: 'babel-loader', options: { presets: ['@babel/preset-typescript', '@babel/preset-react'] } }

// rspack.config.js
{
  loader: 'builtin:swc-loader',
  options: {
    jsc: {
      parser: { syntax: 'typescript', tsx: true },
      transform: { react: { runtime: 'automatic' } },
    },
  },
}
```

---

## Rsbuild — Higher-Level Tool

Rsbuild is a build tool built on Rspack (like Vite is built on Rollup), providing zero-config defaults:

```bash
npm install -D @rsbuild/core
```

```js
// rsbuild.config.ts
import { defineConfig } from '@rsbuild/core';
import { pluginReact } from '@rsbuild/plugin-react';

export default defineConfig({
  plugins: [pluginReact()],
  html: { template: './src/index.html' },
  source: { entry: { index: './src/index.tsx' } },
});
```

---

## Current Limitations (as of 2024)

- Some Webpack plugins not yet compatible (copy-webpack-plugin alternatives needed)
- Older Webpack loader APIs may not work
- Less community tooling and debug support than Webpack
- Some edge cases in HMR differ from Webpack

---

## When to Use Rspack

**Good fit:**
- Large Webpack projects with slow CI builds
- Teams spending significant time waiting for rebuilds
- Projects willing to accept some migration risk

**Not a fit:**
- New projects (use Vite instead — simpler, no Webpack migration needed)
- Projects heavily dependent on unique Webpack plugins
- When Vite already meets your needs

---

## Common Interview Questions

**Q: Why would you choose Rspack over Vite?**
Existing Webpack projects where a full Vite migration is risky. Rspack provides most of Webpack's API compatibility with dramatically better performance. Vite requires rethinking the build architecture; Rspack is a faster drop-in.

**Q: How does Rspack achieve its speed?**
Rust's native parallelism (modules processed on multiple threads), native SWC for TypeScript/JSX transformation (10-100x faster than Babel), and improved module graph algorithms.

**Q: Is Rspack production-ready?**
ByteDance (TikTok) uses it in production for large applications. It's actively maintained with a v1.0 release. Smaller community than Webpack/Vite — check plugin compatibility for your specific use case.
