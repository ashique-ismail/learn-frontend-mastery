# esbuild: Ultra-Fast JavaScript Bundler

## The Idea

**In plain English:** esbuild is a tool that takes all the separate code files that make up a website or app and smashes them together into one (or a few) tidy files that a browser can load super fast. What makes it special is that it does this job 10 to 100 times faster than older tools of the same kind.

**Real-world analogy:** Imagine you are the world's fastest restaurant prep cook. Before the lunch rush, you take every ingredient stored in separate fridges and containers across the kitchen and chop, measure, and arrange them all into ready-to-cook meal kits sitting right on the counter. When an order comes in, the chef grabs the kit and cooks it in seconds instead of hunting for ingredients.

- The ingredients scattered across the kitchen = your JavaScript, TypeScript, and CSS source files spread across many folders
- The prep cook combining them into meal kits = esbuild bundling all those files into a single output file
- The world's fastest prep cook (written in Go, not JavaScript) = why esbuild is so much quicker than older bundlers like Webpack

---

## Overview

esbuild is an extremely fast JavaScript bundler and minifier written in Go. Created by Evan Wallace (co-founder of Figma), esbuild focuses on raw speed and simplicity, achieving bundling speeds 10-100x faster than traditional JavaScript-based bundlers like Webpack and Parcel.

**Key Features:**
- Blazing-fast build performance (written in Go with parallelization)
- Built-in TypeScript and JSX transformation (no external dependencies)
- Tree shaking and code splitting
- Source maps generation
- Minification and compression
- Simple JavaScript/Go/CLI APIs
- ES6 and CommonJS module support
- Plugin architecture for extensibility
- Serves as the foundation for tools like Vite

esbuild trades off some advanced features for incredible speed, making it ideal for library bundling, development builds, and tools that need fast compilation.

## Installation and Setup

### Installation

```bash
# npm
npm install --save-dev esbuild

# yarn
yarn add --dev esbuild

# pnpm
pnpm add --save-dev esbuild

# Global installation
npm install -g esbuild
```

### Basic Usage

```bash
# Bundle a file
esbuild app.js --bundle --outfile=out.js

# Bundle with minification
esbuild app.js --bundle --minify --outfile=out.min.js

# Watch mode for development
esbuild app.js --bundle --outfile=out.js --watch

# Development server
esbuild app.js --bundle --outfile=out.js --servedir=public --serve=8000
```

### Package.json Scripts

```json
{
  "scripts": {
    "build": "esbuild src/index.js --bundle --outfile=dist/bundle.js",
    "build:min": "esbuild src/index.js --bundle --minify --sourcemap --outfile=dist/bundle.min.js",
    "dev": "esbuild src/index.js --bundle --outfile=dist/bundle.js --watch --servedir=public",
    "build:types": "esbuild src/index.ts --bundle --outfile=dist/bundle.js --platform=node --target=node16"
  }
}
```

## Core Concepts

### 1. Native Speed Through Go

esbuild is written in Go, which provides:
- Native compilation to machine code
- Efficient memory usage without garbage collection pauses
- Built-in parallelization using goroutines
- Single binary with no dependencies

**Performance Comparison:**
```
Bundling React application (1,000 modules):
- esbuild: 0.15 seconds
- Webpack 5: 12 seconds
- Parcel 2: 8 seconds
- Rollup: 15 seconds

Speed multiplier: 50-100x faster
```

### 2. Build API Modes

esbuild provides three API modes:

```javascript
// 1. Build API - One-shot bundling
const esbuild = require('esbuild');

esbuild.build({
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'out.js'
}).then(() => console.log('Build complete'));

// 2. Transform API - Single file transformation
const result = esbuild.transformSync('let x: number = 1', {
  loader: 'ts'
});
console.log(result.code); // Transpiled JavaScript

// 3. Serve API - Development server with watch mode
esbuild.serve({
  servedir: 'public',
  port: 8000
}, {
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'public/bundle.js'
});
```

### 3. Loaders and File Types

esbuild automatically handles various file types:

```javascript
const loaders = {
  '.js': 'js',      // JavaScript
  '.jsx': 'jsx',    // React JSX
  '.ts': 'ts',      // TypeScript
  '.tsx': 'tsx',    // TypeScript JSX
  '.json': 'json',  // JSON files
  '.css': 'css',    // CSS
  '.txt': 'text',   // Text files as strings
  '.png': 'dataurl', // Images as data URLs
  '.svg': 'file'    // SVG as file paths
};
```

### 4. Bundling vs Transformation

```javascript
// Bundling: Combines multiple files into one
esbuild.build({
  entryPoints: ['app.js'],
  bundle: true, // Enable bundling
  outfile: 'out.js'
});

// Transformation: Process single file without bundling
const result = esbuild.transformSync(code, {
  loader: 'ts',
  target: 'es2020'
});
```

## API Reference

### Build API

```javascript
const esbuild = require('esbuild');

esbuild.build({
  // Entry points
  entryPoints: ['src/app.js', 'src/admin.js'],
  
  // Or entry points with custom names
  entryPoints: {
    'app': 'src/app.js',
    'admin': 'src/admin.js'
  },
  
  // Output configuration
  outdir: 'dist',
  outfile: 'bundle.js', // Use outfile OR outdir, not both
  outbase: 'src', // Preserve directory structure
  
  // Bundling options
  bundle: true, // Enable bundling
  splitting: true, // Code splitting (requires format: 'esm')
  format: 'esm', // 'iife', 'cjs', 'esm'
  platform: 'browser', // 'browser' or 'node'
  target: ['es2020', 'chrome80', 'firefox78'],
  
  // Minification and optimization
  minify: true,
  minifyWhitespace: true,
  minifyIdentifiers: true,
  minifySyntax: true,
  treeShaking: true,
  
  // Source maps
  sourcemap: true, // Boolean or 'inline', 'external', 'both'
  sourcesContent: true,
  
  // Asset handling
  loader: {
    '.png': 'dataurl',
    '.svg': 'file',
    '.woff': 'file'
  },
  assetNames: 'assets/[name]-[hash]',
  
  // External dependencies (don't bundle)
  external: ['react', 'react-dom'],
  
  // Define constants
  define: {
    'process.env.NODE_ENV': '"production"',
    'DEBUG': 'false'
  },
  
  // Banner/Footer injection
  banner: {
    js: '/* My library v1.0.0 */',
    css: '/* Styles */'
  },
  footer: {
    js: '/* End of bundle */'
  },
  
  // Watch mode
  watch: {
    onRebuild(error, result) {
      if (error) console.error('Build failed:', error);
      else console.log('Build succeeded');
    }
  },
  
  // Incremental builds
  incremental: true,
  
  // Logging
  logLevel: 'info', // 'verbose', 'debug', 'info', 'warning', 'error', 'silent'
  color: true,
  
  // Meta information
  metafile: true // Generate build metadata
}).catch(() => process.exit(1));
```

### Transform API

```javascript
const esbuild = require('esbuild');

// Synchronous transformation
const result = esbuild.transformSync(code, {
  loader: 'tsx',
  target: 'es2020',
  minify: true,
  sourcemap: true,
  define: {
    'process.env.NODE_ENV': '"production"'
  }
});

console.log(result.code); // Transformed code
console.log(result.map); // Source map

// Asynchronous transformation
const asyncResult = await esbuild.transform(code, {
  loader: 'ts',
  format: 'esm'
});
```

### Serve API

```javascript
const esbuild = require('esbuild');

// Start development server
esbuild.serve({
  servedir: 'public',
  port: 8000,
  host: 'localhost',
  onRequest: (args) => {
    console.log(`${args.method} ${args.path} - ${args.status}`);
  }
}, {
  entryPoints: ['src/app.js'],
  bundle: true,
  outfile: 'public/bundle.js',
  sourcemap: true
}).then(server => {
  console.log(`Server running at http://localhost:${server.port}`);
});
```

## Advanced Usage

### 1. TypeScript and JSX Configuration

```javascript
const esbuild = require('esbuild');

// TypeScript with custom tsconfig
esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  outfile: 'dist/bundle.js',
  loader: {
    '.ts': 'ts',
    '.tsx': 'tsx'
  },
  // Note: esbuild doesn't use tsconfig.json for bundling
  // Only for path mapping (via tsconfig-paths plugin)
  target: 'es2020',
  
  // JSX configuration
  jsx: 'transform', // 'transform' or 'preserve'
  jsxFactory: 'React.createElement',
  jsxFragment: 'React.Fragment',
  
  // Or automatic JSX runtime (React 17+)
  jsx: 'automatic',
  jsxImportSource: 'react'
});

// Example TypeScript code
// src/App.tsx
interface Props {
  name: string;
  count: number;
}

export function App({ name, count }: Props) {
  return <div>{name}: {count}</div>;
}
```

### 2. Code Splitting

```javascript
// Code splitting requires ESM format
esbuild.build({
  entryPoints: ['src/home.js', 'src/about.js'],
  bundle: true,
  splitting: true, // Enable code splitting
  format: 'esm', // Required for splitting
  outdir: 'dist',
  chunkNames: 'chunks/[name]-[hash]'
});

// Example with dynamic imports
// src/app.js
async function loadFeature() {
  const { Feature } = await import('./feature.js');
  return new Feature();
}

// Results in:
// dist/app.js (main bundle)
// dist/feature.js (split chunk)
// dist/chunks/shared-[hash].js (shared dependencies)
```

### 3. Plugin Architecture

```javascript
const esbuild = require('esbuild');

// Custom plugin example: Import Markdown as HTML
const markdownPlugin = {
  name: 'markdown',
  setup(build) {
    const fs = require('fs');
    const marked = require('marked');
    
    build.onResolve({ filter: /\.md$/ }, args => ({
      path: require.resolve(args.path, { paths: [args.resolveDir] }),
      namespace: 'markdown'
    }));
    
    build.onLoad({ filter: /.*/, namespace: 'markdown' }, async (args) => {
      const markdown = await fs.promises.readFile(args.path, 'utf8');
      const html = marked.parse(markdown);
      
      return {
        contents: `export default ${JSON.stringify(html)}`,
        loader: 'js'
      };
    });
  }
};

// Use the plugin
esbuild.build({
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'out.js',
  plugins: [markdownPlugin]
});

// Usage in code
import content from './README.md';
document.body.innerHTML = content;
```

### 4. Environment Variables and Define

```javascript
const esbuild = require('esbuild');

esbuild.build({
  entryPoints: ['src/app.js'],
  bundle: true,
  outfile: 'dist/app.js',
  define: {
    // Replace at build time
    'process.env.NODE_ENV': '"production"',
    'process.env.API_URL': '"https://api.example.com"',
    'VERSION': '"1.2.3"',
    '__DEV__': 'false',
    'DEBUG': 'false'
  }
});

// Source code
if (process.env.NODE_ENV === 'production') {
  console.log('Production build');
}
// Becomes: if ("production" === 'production') { ... }
// Dead code elimination removes the condition

if (__DEV__) {
  console.log('Debug info'); // Completely removed in build
}
```

### 5. External Dependencies

```javascript
// Don't bundle certain packages
esbuild.build({
  entryPoints: ['src/index.js'],
  bundle: true,
  outfile: 'dist/bundle.js',
  platform: 'node',
  
  // Mark as external (won't be bundled)
  external: [
    'react',
    'react-dom',
    'lodash',
    './node_modules/*' // All node_modules
  ]
});

// For libraries, use peer dependencies
// package.json
{
  "peerDependencies": {
    "react": ">=17.0.0"
  }
}
```

### 6. Multiple Entry Points and Formats

```javascript
const esbuild = require('esbuild');

// Build library in multiple formats
async function buildLibrary() {
  const commonOptions = {
    entryPoints: ['src/index.ts'],
    bundle: true,
    minify: true,
    sourcemap: true,
    external: ['react', 'react-dom']
  };
  
  // ESM build
  await esbuild.build({
    ...commonOptions,
    format: 'esm',
    outfile: 'dist/index.esm.js'
  });
  
  // CommonJS build
  await esbuild.build({
    ...commonOptions,
    format: 'cjs',
    outfile: 'dist/index.cjs.js'
  });
  
  // IIFE build (for browsers via CDN)
  await esbuild.build({
    ...commonOptions,
    format: 'iife',
    globalName: 'MyLibrary',
    outfile: 'dist/index.browser.js'
  });
}

buildLibrary().catch(() => process.exit(1));
```

### 7. Watch Mode with Incremental Builds

```javascript
const esbuild = require('esbuild');

async function watch() {
  const ctx = await esbuild.context({
    entryPoints: ['src/app.js'],
    bundle: true,
    outfile: 'dist/app.js',
    sourcemap: true,
    plugins: [/* ... */]
  });
  
  // Enable watch mode
  await ctx.watch();
  
  console.log('Watching for changes...');
  
  // Or serve with watch
  await ctx.serve({
    servedir: 'public',
    port: 8000
  });
}

watch().catch(() => process.exit(1));

// Rebuild manually when needed
async function manualRebuild() {
  const ctx = await esbuild.context({
    entryPoints: ['src/app.js'],
    bundle: true,
    outfile: 'dist/app.js',
    incremental: true
  });
  
  // Initial build
  await ctx.rebuild();
  
  // Rebuild on demand
  setTimeout(async () => {
    await ctx.rebuild();
    console.log('Rebuilt!');
  }, 5000);
  
  // Clean up
  ctx.dispose();
}
```

### 8. Meta File Analysis

```javascript
const esbuild = require('esbuild');
const fs = require('fs');

async function analyzeBuild() {
  const result = await esbuild.build({
    entryPoints: ['src/app.js'],
    bundle: true,
    outfile: 'dist/app.js',
    metafile: true,
    minify: true
  });
  
  // Analyze bundle composition
  const meta = result.metafile;
  
  // Get output file sizes
  Object.entries(meta.outputs).forEach(([file, info]) => {
    console.log(`${file}: ${info.bytes} bytes`);
    
    // Show what's in the bundle
    Object.entries(info.inputs).forEach(([input, inputInfo]) => {
      console.log(`  - ${input}: ${inputInfo.bytesInOutput} bytes`);
    });
  });
  
  // Generate HTML visualization
  const text = await esbuild.analyzeMetafile(meta, {
    verbose: true
  });
  console.log(text);
  
  // Save for external analysis
  fs.writeFileSync('meta.json', JSON.stringify(meta));
}

analyzeBuild();
```

## Plugin Development

### Basic Plugin Structure

```javascript
const myPlugin = {
  name: 'my-plugin',
  setup(build) {
    const fs = require('fs');
    
    // Hook into resolution
    build.onResolve({ filter: /^custom:/ }, args => {
      return {
        path: args.path,
        namespace: 'custom-ns'
      };
    });
    
    // Hook into loading
    build.onLoad({ filter: /.*/, namespace: 'custom-ns' }, args => {
      // Load and transform content
      return {
        contents: 'export default "custom content"',
        loader: 'js'
      };
    });
    
    // Hook into build start/end
    build.onStart(() => {
      console.log('Build starting...');
    });
    
    build.onEnd(result => {
      console.log(`Build ended with ${result.errors.length} errors`);
    });
  }
};
```

### Real-World Plugin Examples

```javascript
// 1. SVG React Component Plugin
const svgrPlugin = {
  name: 'svgr',
  setup(build) {
    const fs = require('fs').promises;
    const svgr = require('@svgr/core').transform;
    
    build.onLoad({ filter: /\.svg$/ }, async (args) => {
      const svg = await fs.readFile(args.path, 'utf8');
      const componentCode = await svgr(svg, {
        plugins: ['@svgr/plugin-jsx']
      }, { componentName: 'SvgComponent' });
      
      return {
        contents: componentCode,
        loader: 'jsx'
      };
    });
  }
};

// 2. Environment Variable Plugin
const envPlugin = {
  name: 'env',
  setup(build) {
    const dotenv = require('dotenv');
    const env = dotenv.config().parsed;
    
    build.initialOptions.define = {
      ...build.initialOptions.define,
      ...Object.keys(env).reduce((acc, key) => {
        acc[`process.env.${key}`] = JSON.stringify(env[key]);
        return acc;
      }, {})
    };
  }
};

// 3. Import Alias Plugin
const aliasPlugin = {
  name: 'alias',
  setup(build) {
    const path = require('path');
    
    const aliases = {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components')
    };
    
    build.onResolve({ filter: /^@/ }, args => {
      for (const [alias, aliasPath] of Object.entries(aliases)) {
        if (args.path.startsWith(alias)) {
          return {
            path: path.resolve(aliasPath, args.path.slice(alias.length + 1))
          };
        }
      }
    });
  }
};

// Use plugins
esbuild.build({
  entryPoints: ['src/app.js'],
  bundle: true,
  outfile: 'dist/app.js',
  plugins: [svgrPlugin, envPlugin, aliasPlugin]
});
```

## Performance Optimization

### 1. Parallel Builds

```javascript
const esbuild = require('esbuild');

// Build multiple targets in parallel
async function parallelBuilds() {
  await Promise.all([
    esbuild.build({
      entryPoints: ['src/client.js'],
      bundle: true,
      outfile: 'dist/client.js',
      platform: 'browser'
    }),
    esbuild.build({
      entryPoints: ['src/server.js'],
      bundle: true,
      outfile: 'dist/server.js',
      platform: 'node'
    }),
    esbuild.build({
      entryPoints: ['src/worker.js'],
      bundle: true,
      outfile: 'dist/worker.js',
      platform: 'browser'
    })
  ]);
  
  console.log('All builds complete!');
}

parallelBuilds();
```

### 2. Caching Strategies

```javascript
// Use incremental builds for faster rebuilds
let lastBuildContext;

async function build() {
  if (!lastBuildContext) {
    lastBuildContext = await esbuild.context({
      entryPoints: ['src/app.js'],
      bundle: true,
      outfile: 'dist/app.js',
      incremental: true
    });
  }
  
  await lastBuildContext.rebuild();
}

// Clean up on exit
process.on('beforeExit', () => {
  lastBuildContext?.dispose();
});
```

### 3. Benchmarking

```javascript
async function benchmark() {
  const start = Date.now();
  
  await esbuild.build({
    entryPoints: ['src/app.js'],
    bundle: true,
    outfile: 'dist/app.js',
    minify: true,
    sourcemap: true,
    metafile: true
  });
  
  const duration = Date.now() - start;
  console.log(`Build completed in ${duration}ms`);
  
  return duration;
}

// Run multiple benchmarks
async function runBenchmarks() {
  const times = [];
  for (let i = 0; i < 10; i++) {
    times.push(await benchmark());
  }
  
  const avg = times.reduce((a, b) => a + b) / times.length;
  console.log(`Average: ${avg}ms`);
}
```

## Limitations and Trade-offs

### 1. No Type Checking

```typescript
// esbuild transpiles but doesn't type-check
interface User {
  name: string;
  age: number;
}

function greet(user: User) {
  console.log(user.name);
}

// This will compile but has type error
greet({ name: 'John' }); // Missing 'age' property

// Solution: Run TypeScript separately
// package.json
{
  "scripts": {
    "build": "tsc --noEmit && esbuild src/index.ts --bundle --outfile=dist/bundle.js",
    "typecheck": "tsc --noEmit"
  }
}
```

### 2. Limited CSS Features

```javascript
// esbuild has basic CSS support
esbuild.build({
  entryPoints: ['src/app.js', 'src/styles.css'],
  bundle: true,
  outdir: 'dist',
  loader: {
    '.css': 'css'
  }
});

// But lacks:
// - CSS Modules (no scoping)
// - SASS/LESS compilation (need preprocessor)
// - Advanced PostCSS features

// Solution: Use plugins or preprocess
const postcss = require('esbuild-postcss');

esbuild.build({
  entryPoints: ['src/app.js'],
  bundle: true,
  outfile: 'dist/app.js',
  plugins: [postcss()]
});
```

### 3. No Hot Module Replacement

```javascript
// esbuild serve API provides live reload, not HMR
esbuild.serve({
  servedir: 'public'
}, {
  entryPoints: ['src/app.js'],
  bundle: true,
  outfile: 'public/bundle.js'
});

// Changes trigger full page reload, not module hot swapping
// For HMR, use Vite (which uses esbuild under the hood)
```

## Use Cases

### 1. Library Bundling

```javascript
// Perfect for building npm packages
const esbuild = require('esbuild');

esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  minify: true,
  sourcemap: true,
  target: 'es2020',
  external: ['react', 'react-dom'],
  format: 'esm',
  outfile: 'dist/index.js',
  banner: {
    js: '// My Library v1.0.0\n// MIT License'
  }
});
```

### 2. Command-Line Tools

```javascript
// Build Node.js CLI tools
esbuild.build({
  entryPoints: ['src/cli.ts'],
  bundle: true,
  platform: 'node',
  target: 'node16',
  outfile: 'dist/cli.js',
  banner: {
    js: '#!/usr/bin/env node'
  }
});

// Make executable
const fs = require('fs');
fs.chmodSync('dist/cli.js', 0o755);
```

### 3. Backend Services

```javascript
// Bundle Node.js applications
esbuild.build({
  entryPoints: ['src/server.ts'],
  bundle: true,
  platform: 'node',
  target: 'node18',
  outfile: 'dist/server.js',
  external: ['pg', 'mysql2'], // Don't bundle native modules
  minify: true
});
```

## Common Mistakes

### 1. Not Handling External Modules

```javascript
// Wrong: Bundling everything (including node_modules)
esbuild.build({
  entryPoints: ['src/server.js'],
  bundle: true,
  outfile: 'dist/server.js',
  platform: 'node'
});
// Result: Huge bundle with all dependencies

// Correct: Mark dependencies as external
esbuild.build({
  entryPoints: ['src/server.js'],
  bundle: true,
  outfile: 'dist/server.js',
  platform: 'node',
  external: Object.keys(require('./package.json').dependencies)
});
```

### 2. Forgetting Platform Configuration

```javascript
// Wrong: Default platform is 'browser'
esbuild.build({
  entryPoints: ['server.js'],
  bundle: true,
  outfile: 'dist/server.js'
});
// Result: Node.js built-ins not resolved correctly

// Correct: Specify platform
esbuild.build({
  entryPoints: ['server.js'],
  bundle: true,
  outfile: 'dist/server.js',
  platform: 'node'
});
```

### 3. Not Using Loader for Assets

```javascript
// Wrong: No loader specified for images
esbuild.build({
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'dist/app.js'
});
// import logo from './logo.png' fails

// Correct: Configure loaders
esbuild.build({
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'dist/app.js',
  loader: {
    '.png': 'dataurl',
    '.jpg': 'file',
    '.svg': 'text'
  }
});
```

## Best Practices

### 1. Use Context API for Development

```javascript
const esbuild = require('esbuild');

async function develop() {
  const ctx = await esbuild.context({
    entryPoints: ['src/app.js'],
    bundle: true,
    outfile: 'dist/app.js',
    sourcemap: true
  });
  
  await ctx.watch();
  await ctx.serve({ servedir: 'public', port: 8000 });
  
  console.log('Development server running on http://localhost:8000');
}

develop();
```

### 2. Separate Type Checking

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "npm run typecheck && esbuild src/index.ts --bundle --outfile=dist/bundle.js",
    "dev": "concurrently \"tsc --noEmit --watch\" \"esbuild src/index.ts --bundle --outfile=dist/bundle.js --watch\""
  }
}
```

### 3. Use Metafile for Analysis

```javascript
const esbuild = require('esbuild');

async function build() {
  const result = await esbuild.build({
    entryPoints: ['src/app.js'],
    bundle: true,
    outfile: 'dist/app.js',
    metafile: true,
    minify: true
  });
  
  console.log(await esbuild.analyzeMetafile(result.metafile));
}
```

## Interview Questions

### 1. Why is esbuild so much faster than Webpack?

**Answer:** esbuild is written in Go, a compiled language that runs as native machine code, while Webpack is written in JavaScript which runs in Node.js with garbage collection overhead. Go provides built-in parallelization with goroutines, allowing esbuild to use multiple CPU cores efficiently. esbuild also makes deliberate trade-offs, skipping some advanced features like full type checking and complex transform pipelines in favor of speed. The entire codebase is optimized for performance from the ground up.

### 2. What's the difference between the build, transform, and serve APIs?

**Answer:** The build API bundles entire applications by processing entry points and their dependencies, creating output files. The transform API processes individual pieces of code without bundling, useful for on-the-fly transpilation. The serve API starts a development server that serves files and rebuilds on changes, combining build functionality with an HTTP server. Build is for production bundling, transform for tooling integration, and serve for development workflows.

### 3. Why doesn't esbuild perform TypeScript type checking?

**Answer:** Type checking is inherently slow because it requires building the full type graph and analyzing relationships across files. esbuild focuses on fast transpilation by treating TypeScript as "JavaScript with type annotations to strip away." This deliberate trade-off allows 100x faster compilation. The recommended approach is to run TypeScript's tsc separately for type checking, either in parallel during development or as a pre-build step.

### 4. How does esbuild handle tree shaking?

**Answer:** esbuild performs automatic tree shaking by analyzing ES module imports/exports and identifying unused code. It only works with ES modules (not CommonJS) because ES modules are statically analyzable. esbuild removes unused exports, eliminates dead code branches, and respects the `sideEffects` field in package.json for aggressive optimization. For best results, use ES module format and mark packages as side-effect-free.

### 5. When should you use esbuild vs Vite vs Webpack?

**Answer:** Use esbuild for library bundling, CLI tools, or when you need the absolute fastest builds and can handle the trade-offs. Use Vite for full application development with features like HMR, dev server optimizations, and a rich plugin ecosystem (Vite uses esbuild internally for dependency pre-bundling). Use Webpack when you need maximum configurability, complex custom loaders, or have an established webpack-based workflow. Vite offers the best developer experience for applications, while esbuild excels at library builds.

### 6. How do esbuild plugins work?

**Answer:** esbuild plugins use a hook-based system with `onResolve` and `onLoad` callbacks. During bundling, esbuild calls `onResolve` hooks to determine how to resolve import paths, allowing custom resolution logic. Then it calls `onLoad` hooks to load file contents, enabling custom transformations. Plugins can also hook into `onStart` and `onEnd` for build lifecycle events. This design is simpler than Webpack's loader system but powerful enough for most use cases.

### 7. What are the limitations of esbuild's code splitting?

**Answer:** esbuild's code splitting only works with ESM format output (`format: 'esm'`), not with CommonJS or IIFE. It requires multiple entry points or dynamic imports to create split points. The splitting algorithm is less sophisticated than Rollup's or Webpack's, potentially creating non-optimal chunk boundaries. However, it's much faster and works well for most use cases. For advanced chunking strategies, Rollup or Webpack might be better choices.

## Key Takeaways

1. **Native speed** from Go provides 10-100x faster bundling than JavaScript-based tools
2. **Simple API** with build, transform, and serve modes covers most bundling needs
3. **Zero configuration** for TypeScript, JSX, and modern JavaScript features
4. **No type checking** is a deliberate trade-off for speed; run tsc separately
5. **Plugin system** enables extensibility with onResolve and onLoad hooks
6. **Tree shaking** works automatically with ES modules and side-effect annotations
7. **Code splitting** requires ESM format and multiple entry points or dynamic imports
8. **Perfect for libraries** due to speed and multiple format output support
9. **Limited CSS features** compared to dedicated CSS processors; use plugins for advanced needs
10. **Foundation for Vite** and other modern tools that prioritize developer experience

## Resources

- **Official Documentation**: https://esbuild.github.io/
- **GitHub Repository**: https://github.com/evanw/esbuild
- **Plugin List**: https://github.com/esbuild/community-plugins
- **API Reference**: https://esbuild.github.io/api/
- **Performance Benchmarks**: https://esbuild.github.io/faq/#benchmark-details
- **Awesome esbuild**: Curated list of plugins and resources
- **esbuild Playground**: Online REPL for testing configurations
- **Go Documentation**: Understanding esbuild's language internals
- **Vite Documentation**: See how esbuild integrates into larger tools
- **npm Package**: https://www.npmjs.com/package/esbuild
