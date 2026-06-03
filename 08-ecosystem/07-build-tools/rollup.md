# Rollup: Modern JavaScript Module Bundler

## Overview

Rollup is a module bundler specifically designed for JavaScript libraries and applications. Created by Rich Harris (creator of Svelte), Rollup pioneered tree shaking and ES module bundling, focusing on producing smaller, more efficient bundles by leveraging the static nature of ES modules.

**Key Features:**
- Tree shaking as a first-class feature
- ES module native support with optimal output
- Multiple output formats (ES, CJS, UMD, IIFE, System)
- Plugin-based architecture for extensibility
- Superior code splitting for libraries
- Scope hoisting for better minification
- Simple, predictable output
- Perfect for library authoring
- Minimal runtime overhead

Rollup excels at library bundling and is the foundation for many popular tools including Vite's production build system.

## Installation and Setup

### Installation

```bash
# npm
npm install --save-dev rollup

# yarn
yarn add --dev rollup

# pnpm
pnpm add --save-dev rollup

# Global installation
npm install -g rollup
```

### Basic Project Setup

```bash
# Create project
mkdir my-library && cd my-library
npm init -y
npm install --save-dev rollup

# Create source
mkdir src
touch src/index.js rollup.config.js
```

### Basic Configuration

```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  }
};
```

### Package.json Scripts

```json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "dist/index.cjs.js",
  "module": "dist/index.esm.js",
  "scripts": {
    "build": "rollup --config",
    "watch": "rollup --config --watch",
    "dev": "rollup --config --watch --environment NODE_ENV:development"
  },
  "devDependencies": {
    "rollup": "^4.12.0"
  }
}
```

## Core Concepts

### 1. ES Module-First Design

Rollup was built around ES modules from the start:

```javascript
// ES modules enable static analysis
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export function multiply(a, b) { return a * b; }

// Only imported functions are bundled
import { add, subtract } from './math';
console.log(add(2, 3)); // multiply is tree-shaken

// CommonJS requires runtime analysis
// const math = require('./math'); // Can't tree shake
```

### 2. Tree Shaking

Rollup's killer feature eliminates dead code:

```javascript
// utils.js - Large library
export function used() {
  return 'I am used';
}

export function unused() {
  return 'I am not used';
}

export function alsoUnused() {
  return 'Me neither';
}

// app.js
import { used } from './utils';
console.log(used());

// Output bundle only includes 'used' function
// unused and alsoUnused are completely removed
```

### 3. Scope Hoisting

Rollup merges modules into a single scope when possible:

```javascript
// Before (separate scopes)
// math.js
export const PI = 3.14159;
// app.js
import { PI } from './math';
console.log(PI);

// After Rollup bundling (single scope)
const PI = 3.14159;
console.log(PI);

// Benefits:
// - Smaller bundle size
// - Better minification
// - Faster execution
```

### 4. Output Formats

Rollup supports multiple output formats:

```javascript
// ESM (ES Module) - Modern JavaScript
export { add, subtract };

// CJS (CommonJS) - Node.js
module.exports = { add, subtract };

// UMD (Universal Module Definition) - Browser & Node
(function (global, factory) {
  typeof exports === 'object' ? module.exports = factory() :
  typeof define === 'function' ? define(factory) :
  global.MyLibrary = factory();
}(this, function () { return { add, subtract }; }));

// IIFE (Immediately Invoked Function Expression) - Browser
var MyLibrary = (function () {
  return { add, subtract };
}());

// SystemJS - Dynamic module loader
System.register([], function (exports) {
  return { execute: function () {
    exports({ add, subtract });
  }};
});
```

## Configuration

### Single Output Configuration

```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  
  output: {
    file: 'dist/bundle.js',
    format: 'esm',
    sourcemap: true,
    name: 'MyLibrary' // For IIFE/UMD
  },
  
  // External dependencies (not bundled)
  external: ['react', 'react-dom'],
  
  // Plugins
  plugins: []
};
```

### Multiple Output Configurations

```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  
  output: [
    // ES module build
    {
      file: 'dist/index.esm.js',
      format: 'esm',
      sourcemap: true
    },
    // CommonJS build
    {
      file: 'dist/index.cjs.js',
      format: 'cjs',
      sourcemap: true,
      exports: 'auto' // or 'named', 'default', 'none'
    },
    // UMD build for browsers
    {
      file: 'dist/index.umd.js',
      format: 'umd',
      name: 'MyLibrary',
      sourcemap: true,
      globals: {
        react: 'React',
        'react-dom': 'ReactDOM'
      }
    }
  ],
  
  external: ['react', 'react-dom']
};
```

### Multiple Entry Points

```javascript
// rollup.config.js
export default [
  // Main library build
  {
    input: 'src/index.js',
    output: {
      file: 'dist/index.js',
      format: 'esm'
    }
  },
  // CLI tool build
  {
    input: 'src/cli.js',
    output: {
      file: 'dist/cli.js',
      format: 'cjs',
      banner: '#!/usr/bin/env node'
    }
  },
  // Browser bundle
  {
    input: 'src/browser.js',
    output: {
      file: 'dist/browser.js',
      format: 'iife',
      name: 'MyLib'
    }
  }
];
```

## Essential Plugins

### 1. @rollup/plugin-node-resolve

Resolves node_modules dependencies:

```bash
npm install --save-dev @rollup/plugin-node-resolve
```

```javascript
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  plugins: [
    resolve({
      // Use Node resolution algorithm
      preferBuiltins: true,
      // Browser-specific resolution
      browser: true,
      // Include dev dependencies
      modulesOnly: false,
      // Extensions to resolve
      extensions: ['.js', '.json', '.node']
    })
  ]
};
```

### 2. @rollup/plugin-commonjs

Converts CommonJS to ES modules:

```bash
npm install --save-dev @rollup/plugin-commonjs
```

```javascript
import commonjs from '@rollup/plugin-commonjs';
import resolve from '@rollup/plugin-node-resolve';

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  plugins: [
    resolve(),
    commonjs({
      // Include node_modules
      include: 'node_modules/**',
      // Convert CommonJS require to ES imports
      transformMixedEsModules: true,
      // Ignore specific modules
      ignore: ['conditional-runtime-dependency']
    })
  ]
};

// Now you can import CommonJS modules
import _ from 'lodash'; // CommonJS package
import React from 'react'; // ES module
```

### 3. @rollup/plugin-typescript

TypeScript support:

```bash
npm install --save-dev @rollup/plugin-typescript typescript tslib
```

```javascript
import typescript from '@rollup/plugin-typescript';

export default {
  input: 'src/index.ts',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  plugins: [
    typescript({
      tsconfig: './tsconfig.json',
      declaration: true,
      declarationDir: './dist/types'
    })
  ]
};

// src/index.ts
export interface User {
  id: number;
  name: string;
}

export function greetUser(user: User): string {
  return `Hello, ${user.name}!`;
}
```

### 4. @rollup/plugin-babel

Transpile modern JavaScript:

```bash
npm install --save-dev @rollup/plugin-babel @babel/core @babel/preset-env
```

```javascript
import babel from '@rollup/plugin-babel';

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  plugins: [
    babel({
      babelHelpers: 'bundled',
      exclude: 'node_modules/**',
      presets: [
        ['@babel/preset-env', {
          targets: '> 0.5%, not dead'
        }]
      ]
    })
  ]
};
```

### 5. @rollup/plugin-terser

Minification for production:

```bash
npm install --save-dev @rollup/plugin-terser
```

```javascript
import terser from '@rollup/plugin-terser';

export default {
  input: 'src/index.js',
  output: [
    // Unminified
    {
      file: 'dist/bundle.js',
      format: 'esm'
    },
    // Minified
    {
      file: 'dist/bundle.min.js',
      format: 'esm',
      plugins: [terser({
        compress: {
          drop_console: true,
          dead_code: true
        },
        mangle: true,
        format: {
          comments: false
        }
      })]
    }
  ]
};
```

### 6. @rollup/plugin-json

Import JSON files:

```bash
npm install --save-dev @rollup/plugin-json
```

```javascript
import json from '@rollup/plugin-json';

export default {
  input: 'src/index.js',
  plugins: [json()],
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  }
};

// src/index.js
import pkg from '../package.json';
console.log(`Version: ${pkg.version}`);
```

## Advanced Usage

### 1. Code Splitting

```javascript
// rollup.config.js
export default {
  input: {
    main: 'src/main.js',
    admin: 'src/admin.js'
  },
  output: {
    dir: 'dist',
    format: 'esm',
    // Control chunk naming
    chunkFileNames: 'chunks/[name]-[hash].js',
    entryFileNames: '[name].js'
  },
  // Manual chunks for better control
  manualChunks: {
    vendor: ['react', 'react-dom'],
    utils: ['lodash', 'date-fns']
  }
};

// Dynamic imports create automatic split points
// src/main.js
async function loadAdmin() {
  const { Admin } = await import('./admin-panel');
  return new Admin();
}

// Results in:
// dist/main.js
// dist/admin.js
// dist/chunks/admin-panel-[hash].js
// dist/chunks/vendor-[hash].js
```

### 2. Watch Mode for Development

```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  watch: {
    include: 'src/**',
    exclude: 'node_modules/**',
    clearScreen: false,
    // Custom watch options
    buildDelay: 100,
    chokidar: {
      // Chokidar options
      usePolling: true
    }
  }
};

// Run watch mode
// npm run watch
```

### 3. Environment-Specific Builds

```javascript
// rollup.config.js
import replace from '@rollup/plugin-replace';
import terser from '@rollup/plugin-terser';

const production = process.env.NODE_ENV === 'production';

export default {
  input: 'src/index.js',
  output: {
    file: production ? 'dist/bundle.min.js' : 'dist/bundle.js',
    format: 'esm',
    sourcemap: !production
  },
  plugins: [
    replace({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      preventAssignment: true
    }),
    production && terser()
  ].filter(Boolean)
};

// Usage in code
if (process.env.NODE_ENV === 'development') {
  console.log('Debug mode');
}
// In production: if ('production' === 'development') -> eliminated
```

### 4. Custom Plugin Development

```javascript
// rollup-plugin-banner.js
export default function banner(options = {}) {
  return {
    name: 'banner',
    
    // Hook into bundle generation
    renderChunk(code, chunk, options) {
      const banner = `/**
 * ${options.title || 'My Library'}
 * @version ${options.version || '1.0.0'}
 * @license ${options.license || 'MIT'}
 */`;
      
      return {
        code: banner + '\n' + code,
        map: null
      };
    }
  };
}

// rollup.config.js
import banner from './rollup-plugin-banner';
import pkg from './package.json';

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  plugins: [
    banner({
      title: pkg.name,
      version: pkg.version,
      license: pkg.license
    })
  ]
};
```

### 5. Advanced Plugin Hooks

```javascript
// Custom plugin with multiple hooks
export default function advancedPlugin() {
  return {
    name: 'advanced-plugin',
    
    // Build hooks (sequential)
    buildStart(options) {
      console.log('Build starting...');
    },
    
    resolveId(source, importer) {
      if (source.startsWith('virtual:')) {
        return source; // Mark as virtual module
      }
      return null; // Use default resolution
    },
    
    load(id) {
      if (id.startsWith('virtual:')) {
        return `export default "virtual content"`;
      }
      return null;
    },
    
    transform(code, id) {
      if (id.endsWith('.custom')) {
        return {
          code: transformCustom(code),
          map: null
        };
      }
      return null;
    },
    
    // Output generation hooks
    renderStart(outputOptions, inputOptions) {
      console.log('Rendering output...');
    },
    
    renderChunk(code, chunk) {
      // Modify chunk code
      return {
        code: code.replace(/DEBUG/g, 'false'),
        map: null
      };
    },
    
    generateBundle(options, bundle) {
      // Access full bundle
      for (const [fileName, chunk] of Object.entries(bundle)) {
        console.log(`Generated: ${fileName}`);
      }
    },
    
    buildEnd(error) {
      if (error) {
        console.error('Build failed:', error);
      } else {
        console.log('Build complete!');
      }
    }
  };
}
```

### 6. Library Authoring Best Practices

```javascript
// rollup.config.js - Complete library setup
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import typescript from '@rollup/plugin-typescript';
import terser from '@rollup/plugin-terser';
import dts from 'rollup-plugin-dts';
import pkg from './package.json';

const production = process.env.NODE_ENV === 'production';

export default [
  // Main library builds
  {
    input: 'src/index.ts',
    output: [
      // ESM build
      {
        file: pkg.module,
        format: 'esm',
        sourcemap: true
      },
      // CommonJS build
      {
        file: pkg.main,
        format: 'cjs',
        sourcemap: true,
        exports: 'auto'
      },
      // UMD build for CDN
      {
        file: pkg.browser,
        format: 'umd',
        name: 'MyLibrary',
        sourcemap: true,
        globals: {
          react: 'React'
        }
      }
    ],
    external: [
      ...Object.keys(pkg.peerDependencies || {}),
      ...Object.keys(pkg.dependencies || {})
    ],
    plugins: [
      resolve(),
      commonjs(),
      typescript({
        tsconfig: './tsconfig.json',
        declaration: false // Generated separately
      }),
      production && terser()
    ].filter(Boolean)
  },
  
  // TypeScript declarations
  {
    input: 'src/index.ts',
    output: {
      file: pkg.types,
      format: 'esm'
    },
    plugins: [dts()]
  }
];

// package.json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "dist/index.cjs.js",
  "module": "dist/index.esm.js",
  "browser": "dist/index.umd.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.cjs.js",
      "import": "./dist/index.esm.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist"],
  "sideEffects": false
}
```

## Tree Shaking Optimization

### Best Practices for Tree Shaking

```javascript
// 1. Use ES modules, not CommonJS
// Good: ES module exports
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }

// Bad: CommonJS (can't tree shake)
// module.exports = { add, subtract };

// 2. Avoid side effects in modules
// Good: Pure functions
export function calculate(x) {
  return x * 2;
}

// Bad: Side effects prevent tree shaking
console.log('Module loaded'); // Side effect!
export function calculate(x) {
  return x * 2;
}

// 3. Mark packages as side-effect-free
// package.json
{
  "sideEffects": false
  // Or specify files with side effects
  "sideEffects": ["*.css", "*.scss"]
}

// 4. Use named imports
// Good: Enables tree shaking
import { add } from './math';

// Bad: Imports everything
import * as math from './math';
console.log(math.add(1, 2));

// 5. Avoid default exports for libraries
// Good: Named exports allow selective imports
export function add() {}
export function subtract() {}

// Less ideal: Default export bundles everything
export default { add, subtract };
```

## Code Splitting Strategies

### Entry-Based Splitting

```javascript
export default {
  input: {
    app: 'src/app.js',
    admin: 'src/admin.js',
    worker: 'src/worker.js'
  },
  output: {
    dir: 'dist',
    format: 'esm',
    entryFileNames: '[name].js',
    chunkFileNames: 'chunks/[name]-[hash].js'
  }
};
```

### Dynamic Import Splitting

```javascript
// Automatic code splitting with dynamic imports
async function loadFeature(featureName) {
  if (featureName === 'charts') {
    const { Chart } = await import('./features/charts');
    return new Chart();
  } else if (featureName === 'tables') {
    const { Table } = await import('./features/tables');
    return new Table();
  }
}

// Creates separate chunks for charts and tables
```

### Manual Chunk Assignment

```javascript
export default {
  input: 'src/index.js',
  output: {
    dir: 'dist',
    format: 'esm'
  },
  manualChunks(id) {
    // Vendor chunk for node_modules
    if (id.includes('node_modules')) {
      return 'vendor';
    }
    
    // Utility chunk
    if (id.includes('src/utils')) {
      return 'utils';
    }
    
    // Component chunks by directory
    if (id.includes('src/components/charts')) {
      return 'charts';
    }
  }
};
```

## Common Mistakes

### 1. Not Marking External Dependencies

```javascript
// Wrong: Bundles everything including React
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  }
};
// Result: 150KB bundle with React included

// Correct: Mark peer dependencies as external
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  external: ['react', 'react-dom']
};
// Result: 5KB bundle, React used from consumer
```

### 2. Wrong Plugin Order

```javascript
// Wrong: CommonJS plugin before resolve
export default {
  plugins: [
    commonjs(), // Too early!
    resolve()
  ]
};

// Correct: Resolve first, then CommonJS
export default {
  plugins: [
    resolve(), // Find modules first
    commonjs() // Then convert them
  ]
};
```

### 3. Missing Globals for UMD

```javascript
// Wrong: External libs not mapped to globals
export default {
  output: {
    format: 'umd',
    name: 'MyLib'
  },
  external: ['react']
};
// Error: 'react' is not defined

// Correct: Map externals to globals
export default {
  output: {
    format: 'umd',
    name: 'MyLib',
    globals: {
      react: 'React',
      'react-dom': 'ReactDOM'
    }
  },
  external: ['react', 'react-dom']
};
```

### 4. Side Effects Blocking Tree Shaking

```javascript
// Wrong: Side effects in module
console.log('Initializing...'); // Side effect!

export function utility() {
  return 'value';
}
// Can't tree shake even if unused

// Correct: No side effects
export function utility() {
  return 'value';
}
// Can be tree-shaken if unused
```

## Best Practices

### 1. Optimize for Library Consumption

```javascript
// Multi-format builds
export default {
  input: 'src/index.js',
  output: [
    { file: 'dist/index.esm.js', format: 'esm' },
    { file: 'dist/index.cjs.js', format: 'cjs' },
    { file: 'dist/index.umd.js', format: 'umd', name: 'MyLib' }
  ],
  external: Object.keys(pkg.peerDependencies)
};
```

### 2. Use Source Maps

```javascript
export default {
  output: {
    file: 'dist/bundle.js',
    format: 'esm',
    sourcemap: true // Enable source maps
  }
};
```

### 3. Leverage Caching

```javascript
// Use watch mode with cache
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm'
  },
  cache: true, // Enable caching
  watch: {
    include: 'src/**'
  }
};
```

### 4. Document Package Exports

```json
// package.json
{
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.cjs.js",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "import": "./dist/utils.esm.js",
      "require": "./dist/utils.cjs.js"
    }
  }
}
```

## When to Use Rollup

### Ideal Use Cases

1. **Library Development**: Rollup excels at creating npm packages
2. **Framework Core**: Many frameworks use Rollup (Svelte, Vue)
3. **Multiple Output Formats**: Easy ESM, CJS, UMD generation
4. **Tree Shaking Priority**: Optimal dead code elimination
5. **Clean Output**: Predictable, readable bundle code

### When to Consider Alternatives

```javascript
// Use Vite for:
const useVite = [
  'Full application development',
  'Hot Module Replacement needs',
  'Development server with proxying',
  'Frontend framework projects'
];

// Use Webpack for:
const useWebpack = [
  'Complex applications with many features',
  'Need extensive loaders and plugins',
  'Module federation',
  'Legacy project requirements'
];

// Use esbuild for:
const useEsbuild = [
  'Speed is absolute priority',
  'Simple builds without advanced features',
  'CLI tools and Node.js services',
  'Minimal configuration preference'
];
```

## Interview Questions

### 1. What is tree shaking and how does Rollup implement it?

**Answer:** Tree shaking eliminates unused code from bundles by analyzing ES module imports/exports. Rollup pioneered this technique by leveraging the static structure of ES modules to determine which exports are actually used. During bundling, Rollup marks all imported values, then removes any unmarked (unused) code. This only works with ES modules because they're statically analyzable, unlike CommonJS which uses dynamic require() calls. Rollup's implementation is particularly effective because it performs live code inclusion rather than dead code elimination.

### 2. Why is Rollup preferred for library development?

**Answer:** Rollup produces cleaner, more efficient output with minimal runtime overhead, making it ideal for libraries. It supports multiple output formats (ESM, CJS, UMD) from a single source, crucial for library distribution. Rollup's tree shaking ensures consumers only bundle what they use. Scope hoisting reduces bundle size and improves performance. The predictable output makes debugging easier. Finally, Rollup is designed for ES modules, which are the future standard for JavaScript modules.

### 3. Explain Rollup's scope hoisting and its benefits.

**Answer:** Scope hoisting merges all modules into a single scope when possible, rather than wrapping each module in a function. This reduces bundle size by eliminating wrapper functions, enables better minification by allowing variable name mangling across modules, and improves runtime performance by reducing function call overhead. For example, instead of wrapping each module in `(function(){...})()`, Rollup concatenates them directly, using rename strategies to avoid conflicts.

### 4. How does Rollup handle code splitting?

**Answer:** Rollup supports code splitting through multiple entry points and dynamic imports. With multiple entries, Rollup automatically extracts shared dependencies into separate chunks. Dynamic imports create split points that generate separate chunks loaded on demand. The `manualChunks` option provides fine-grained control over chunk creation. Rollup's algorithm optimizes chunk boundaries to minimize duplication while maximizing parallel loading opportunities.

### 5. What's the difference between Rollup and Webpack for bundling?

**Answer:** Rollup focuses on ES modules and optimal library bundling with clean output and superior tree shaking. It's simpler, with fewer features but more predictable results. Webpack is application-focused with extensive features like HMR, code splitting, asset management, and a vast plugin ecosystem. Webpack handles all asset types, while Rollup primarily handles JavaScript. Rollup produces cleaner code suitable for libraries; Webpack is better for complex applications with diverse asset requirements.

### 6. How do Rollup plugins work?

**Answer:** Rollup plugins use a hook-based system that taps into different build phases. Key hooks include `resolveId` for module resolution, `load` for reading file contents, `transform` for code transformation, and `generateBundle` for output manipulation. Plugins return objects with hook functions that Rollup calls at appropriate times. The plugin architecture is simpler than Webpack's loader/plugin split, with a clear execution order and well-defined responsibilities for each hook.

### 7. What are the trade-offs of using Rollup?

**Answer:** Rollup excels at JavaScript bundling but has limited asset handling compared to Webpack. It requires plugins for basic features like CommonJS compatibility and Node module resolution. The development experience lacks built-in HMR and dev server. Configuration can become complex for application builds with many asset types. However, for libraries, these trade-offs are acceptable because Rollup's strengths (clean output, tree shaking, multiple formats) are exactly what library authors need.

## Key Takeaways

1. **Tree shaking pioneer** with live code inclusion for optimal dead code elimination
2. **ES module native** design enables superior static analysis and optimization
3. **Multiple output formats** (ESM, CJS, UMD, IIFE) from single source for libraries
4. **Scope hoisting** merges modules for smaller bundles and better performance
5. **Clean output** with minimal runtime overhead and predictable code generation
6. **Library-focused** design makes it the standard for npm package bundling
7. **Plugin architecture** provides extensibility through well-defined hooks
8. **Code splitting** supports both automatic and manual chunk creation strategies
9. **Predictable builds** with simpler configuration than application bundlers
10. **Foundation for Vite** production builds and other modern tooling

## Resources

- **Official Documentation**: https://rollupjs.org/
- **GitHub Repository**: https://github.com/rollup/rollup
- **Plugin List**: https://github.com/rollup/awesome
- **Tutorial**: https://rollupjs.org/tutorial/
- **Configuration Guide**: https://rollupjs.org/configuration-options/
- **Plugin Development**: https://rollupjs.org/plugin-development/
- **Tree Shaking Guide**: Deep dive into optimization techniques
- **Vite Documentation**: See Rollup in production use
- **Rich Harris Blog**: Creator's insights on Rollup design
- **npm Library Examples**: Study popular libraries using Rollup
