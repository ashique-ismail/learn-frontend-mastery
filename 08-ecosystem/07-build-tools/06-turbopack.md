# Turbopack: Next-Generation Bundler for Next.js

## The Idea

**In plain English:** Turbopack is a tool that takes all the separate files of your website (code, styles, images) and bundles them together into a fast, browser-ready package — and it does this so quickly that when you change one file, your browser updates in milliseconds. A "bundler" is just a program that combines many files into fewer, optimized ones so websites load fast.

**Real-world analogy:** Imagine a school cafeteria kitchen that prepares hundreds of individual meal components (proteins, veggies, sauces) for thousands of students. A slow kitchen re-cooks everything from scratch every time one ingredient changes. A smart kitchen like Turbopack only re-cooks the dish that actually changed, keeps everything else warm and ready, and has multiple chefs working in parallel.
- The kitchen = Turbopack (the bundler)
- Each dish component = a source file in your project
- Re-cooking only the changed dish = incremental bundling (only rebuilding what changed)
- Multiple chefs working at once = parallel processing powered by Rust

---

## Overview

Turbopack is a next-generation JavaScript bundler written in Rust, designed as the successor to Webpack in the Next.js ecosystem. Created by Vercel (the team behind Next.js), Turbopack leverages incremental computation and low-level optimizations to deliver blazing-fast build performance, especially for large-scale applications.

**Key Features:**
- Incremental bundling architecture with persistent caching
- Hot Module Replacement (HMR) with sub-second updates
- Native support for TypeScript, JSX, and modern JavaScript
- Built-in support for CSS, CSS Modules, and PostCSS
- Tree shaking and code splitting optimizations
- Native image optimization integration
- Designed specifically for Next.js 13+ workflows

Turbopack aims to solve the performance bottlenecks that plague traditional bundlers as applications scale, providing consistent fast builds regardless of project size.

## Installation and Setup

### Next.js Integration (Recommended)

Turbopack is integrated into Next.js 13+ and can be enabled with a flag:

```bash
# Create a new Next.js project
npx create-next-app@latest my-app --typescript

# Navigate to project
cd my-app

# Run with Turbopack (development mode)
npm run dev -- --turbo
# or
yarn dev --turbo
# or
pnpm dev --turbo
```

### Package.json Configuration

Update your `package.json` scripts to use Turbopack by default:

```json
{
  "scripts": {
    "dev": "next dev --turbo",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

**Note:** As of 2026, Turbopack is stable for Next.js development mode and is gradually rolling out for production builds.

### Standalone Usage (Experimental)

For non-Next.js projects, Turbopack can be used via the CLI (experimental):

```bash
# Install Turbopack CLI
npm install --save-dev turbopack

# Run development server
npx turbopack dev

# Build for production
npx turbopack build
```

## Core Concepts

### 1. Incremental Bundling Architecture

Turbopack uses an incremental computation model where only changed files and their dependents are rebundled.

**Traditional Bundler Flow:**
```
Code Change → Full Bundle Rebuild → Update Browser
(Time: seconds to minutes for large apps)
```

**Turbopack Incremental Flow:**
```
Code Change → Compute Affected Modules → Update Only Changed Chunks → HMR Update
(Time: milliseconds)
```

### 2. Rust-Based Performance

Turbopack is written in Rust, providing:
- Memory safety without garbage collection overhead
- Parallel processing with fearless concurrency
- Zero-cost abstractions for optimal performance
- Native speed comparable to Go-based tools like esbuild

### 3. Function-Level Caching

Turbopack caches compilation results at the function level, not just file level:

```javascript
// If only the Button component changes, only Button is recompiled
export function Button() { /* changed */ }
export function Header() { /* unchanged - cached */ }
export function Footer() { /* unchanged - cached */ }
```

### 4. Smart Module Graph

Turbopack builds a detailed dependency graph that tracks:
- Direct and transitive dependencies
- CSS dependencies and imports
- Dynamic imports and code splits
- Asset dependencies (images, fonts)

## Basic Configuration

### turbopack.config.js

Create a `turbopack.config.js` file for custom configuration:

```javascript
module.exports = {
  // Entry points for bundling
  entry: {
    main: './src/index.tsx',
    admin: './src/admin.tsx'
  },

  // Output configuration
  output: {
    path: './dist',
    filename: '[name].[hash].js'
  },

  // Module resolution
  resolve: {
    extensions: ['.tsx', '.ts', '.jsx', '.js'],
    alias: {
      '@components': './src/components',
      '@utils': './src/utils'
    }
  },

  // Environment variables
  env: {
    API_URL: process.env.API_URL,
    ENABLE_ANALYTICS: process.env.NODE_ENV === 'production'
  }
};
```

### TypeScript Support

Turbopack has native TypeScript support with zero configuration:

```typescript
// tsconfig.json - automatically detected
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

### CSS and Styling

Turbopack supports multiple CSS approaches out of the box:

```javascript
// 1. CSS Modules (automatic)
import styles from './Button.module.css';

export function Button() {
  return <button className={styles.primary}>Click me</button>;
}

// 2. Global CSS
import './globals.css';

// 3. CSS-in-JS (with proper plugins)
import styled from 'styled-components';

const StyledButton = styled.button`
  background: blue;
  color: white;
`;

// 4. Tailwind CSS (via PostCSS)
export function Card() {
  return <div className="rounded-lg shadow-md p-4">Content</div>;
}
```

## Advanced Usage

### 1. Code Splitting and Dynamic Imports

Turbopack automatically optimizes code splitting:

```typescript
// Automatic code splitting with dynamic imports
import dynamic from 'next/dynamic';

// Component-level splitting
const DynamicChart = dynamic(() => import('@/components/Chart'), {
  loading: () => <p>Loading chart...</p>,
  ssr: false // Client-side only
});

// Route-level splitting
const AdminPanel = dynamic(() => import('@/components/AdminPanel'));

// Conditional loading based on feature flags
async function loadFeature(featureName: string) {
  if (featureName === 'analytics') {
    const { Analytics } = await import('@/features/analytics');
    return Analytics;
  } else if (featureName === 'dashboard') {
    const { Dashboard } = await import('@/features/dashboard');
    return Dashboard;
  }
}

// Usage in component
export default function Page() {
  return (
    <div>
      <DynamicChart data={chartData} />
      <AdminPanel />
    </div>
  );
}
```

### 2. Asset Optimization

Turbopack integrates with Next.js Image component for optimal asset handling:

```typescript
import Image from 'next/image';

export function ProductCard({ product }) {
  return (
    <div>
      {/* Automatic optimization, lazy loading, and responsive images */}
      <Image
        src={product.image}
        alt={product.name}
        width={400}
        height={300}
        priority={product.featured} // Preload featured images
        placeholder="blur"
        blurDataURL={product.blurHash}
      />
      
      {/* SVG imports work seamlessly */}
      <img src="/icons/star.svg" alt="Star" />
      
      {/* Font optimization via next/font */}
      <h2 className={inter.className}>{product.name}</h2>
    </div>
  );
}
```

### 3. Environment Variables

Turbopack handles environment variables with Next.js conventions:

```javascript
// .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgresql://localhost:5432/db
SECRET_KEY=super-secret

// Usage in code
// Client-side (NEXT_PUBLIC_ prefix required)
const apiUrl = process.env.NEXT_PUBLIC_API_URL;

// Server-side (any variable accessible)
export async function getServerSideProps() {
  const db = await connectDB(process.env.DATABASE_URL);
  const data = await db.query();
  
  return {
    props: {
      data
    }
  };
}
```

### 4. Custom Loaders and Transformers

Configure custom file transformations:

```javascript
// next.config.js
module.exports = {
  experimental: {
    turbo: {
      loaders: {
        // Custom loader for .md files
        '.md': ['markdown-loader'],
        
        // SVG as React components
        '.svg': ['@svgr/webpack'],
        
        // YAML data files
        '.yaml': ['yaml-loader']
      },
      
      resolveAlias: {
        '@': './src',
        '@components': './src/components',
        '@utils': './src/utils'
      },
      
      resolveExtensions: [
        '.tsx',
        '.ts',
        '.jsx',
        '.js',
        '.json'
      ]
    }
  }
};
```

### 5. Tree Shaking Optimization

Turbopack automatically tree-shakes unused code:

```typescript
// utils.ts - Large utility library
export function usedFunction() { /* ... */ }
export function unusedFunction() { /* ... */ }
export function anotherUnused() { /* ... */ }

// app.tsx - Only import what you need
import { usedFunction } from './utils';

// Turbopack automatically removes unusedFunction and anotherUnused
// from the final bundle

// Ensure tree shaking works with side-effect-free packages
// package.json in your library
{
  "name": "my-utils",
  "sideEffects": false, // Enables aggressive tree shaking
  // or specify files with side effects
  "sideEffects": ["*.css", "*.scss"]
}
```

### 6. Monorepo Support

Turbopack works seamlessly in monorepos:

```javascript
// apps/web/next.config.js
module.exports = {
  experimental: {
    turbo: {
      resolveExtensions: ['.tsx', '.ts', '.jsx', '.js'],
      
      // Reference workspace packages
      resolveAlias: {
        '@repo/ui': '../../packages/ui/src',
        '@repo/utils': '../../packages/utils/src'
      }
    }
  },
  
  // Transpile workspace packages
  transpilePackages: ['@repo/ui', '@repo/utils']
};

// Usage in app
import { Button } from '@repo/ui';
import { formatDate } from '@repo/utils';

export default function Page() {
  return (
    <div>
      <Button>Click me</Button>
      <p>{formatDate(new Date())}</p>
    </div>
  );
}
```

## HMR Performance Benchmarks

### Turbopack vs Webpack 5 vs Vite

Based on Vercel's benchmarks (2026 data) for a large Next.js application:

```
Application with 3,000 modules:

Cold Start (Initial Compile):
- Turbopack: 1.2s
- Vite: 2.1s
- Webpack 5: 16.5s

Hot Module Replacement (File Save to Browser Update):
- Turbopack: 8ms
- Vite: 45ms
- Webpack 5: 1,200ms

Application with 30,000 modules:

Cold Start:
- Turbopack: 2.8s
- Vite: 8.4s
- Webpack 5: 98s

HMR:
- Turbopack: 12ms
- Vite: 180ms
- Webpack 5: 4,500ms
```

### Real-World Performance Example

```typescript
// Measure HMR performance in development
export default function DevMetrics() {
  const [lastUpdate, setLastUpdate] = React.useState(Date.now());
  
  React.useEffect(() => {
    // Track when component re-renders after code change
    const updateTime = Date.now() - lastUpdate;
    console.log(`HMR update time: ${updateTime}ms`);
    setLastUpdate(Date.now());
  }, [lastUpdate]);
  
  return (
    <div>
      <p>Last update: {new Date(lastUpdate).toISOString()}</p>
      {/* Make changes to this component and observe console */}
    </div>
  );
}
```

## Comparison with Other Bundlers

### Turbopack vs Webpack 5

```typescript
// Feature comparison

// 1. Configuration
// Webpack 5: Complex, requires plugins
const webpack = require('webpack');
module.exports = {
  entry: './src/index.js',
  output: { filename: 'bundle.js' },
  module: {
    rules: [
      { test: /\.tsx?$/, use: 'ts-loader' },
      { test: /\.css$/, use: ['style-loader', 'css-loader'] }
    ]
  },
  plugins: [new webpack.HotModuleReplacementPlugin()]
};

// Turbopack: Zero config for Next.js
// Just run: next dev --turbo

// 2. HMR Speed
// Webpack: Rebuilds entire module graph for changes
// Turbopack: Incremental updates only affected modules

// 3. Memory Usage
// Webpack: ~500MB for large apps
// Turbopack: ~150MB for same app (3x more efficient)
```

### Turbopack vs Vite

```typescript
// 1. Bundling Strategy
// Vite: ESM-based, unbundled in dev, Rollup in prod
// Turbopack: Bundled in both dev and prod (but incremental)

// 2. Browser Compatibility
// Vite: Requires modern browser with native ESM
import { feature } from './module.js'; // Browser fetches directly

// Turbopack: Works with all browsers (transpiles as needed)
import { feature } from './module'; // Bundled and optimized

// 3. Large Dependencies
// Vite: Pre-bundles node_modules with esbuild
// Turbopack: Incremental bundling handles large deps efficiently

// 4. Performance Scale
// Vite: Excellent for small-to-medium apps
// Turbopack: Maintains performance at very large scale (30k+ modules)
```

### Turbopack vs esbuild

```typescript
// 1. Feature Completeness
// esbuild: Minimal, focuses on speed
// Turbopack: Full-featured (HMR, code splitting, etc.)

// 2. Use Case
// esbuild: Library bundling, simple builds
import * as esbuild from 'esbuild';
await esbuild.build({
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'out.js'
});

// Turbopack: Full application development with Next.js
// Includes routing, SSR, API routes, etc.

// 3. Extensibility
// esbuild: Plugin API available but limited
// Turbopack: Rich ecosystem via Next.js plugins
```

## Migration Guide

### From Webpack to Turbopack

```javascript
// 1. Update Next.js to 13+
npm install next@latest react@latest react-dom@latest

// 2. Update package.json scripts
{
  "scripts": {
    "dev": "next dev --turbo", // Add --turbo flag
    "build": "next build",
    "start": "next start"
  }
}

// 3. Convert webpack config to next.config.js
// Before (webpack.config.js)
module.exports = {
  module: {
    rules: [
      {
        test: /\.svg$/,
        use: ['@svgr/webpack']
      }
    ]
  }
};

// After (next.config.js)
module.exports = {
  experimental: {
    turbo: {
      loaders: {
        '.svg': ['@svgr/webpack']
      }
    }
  }
};

// 4. Handle webpack-specific code
// Before
if (typeof window === 'undefined') {
  require.context('./images', false, /\.png$/);
}

// After (use dynamic imports)
if (typeof window === 'undefined') {
  const images = import.meta.glob('./images/*.png');
}
```

### From Vite to Turbopack

```typescript
// 1. Replace Vite with Next.js
// Remove vite.config.ts and package.json scripts

// 2. Convert entry point
// Before (Vite index.html)
// <!DOCTYPE html>
// <html>
//   <body>
//     <div id="root"></div>
//     <script type="module" src="/src/main.tsx"></script>
//   </body>
// </html>

// After (Next.js pages/_app.tsx)
import type { AppProps } from 'next/app';
import '../styles/globals.css';

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}

// 3. Convert Vite env variables
// Before (.env)
// VITE_API_URL=https://api.example.com

// After (.env.local)
// NEXT_PUBLIC_API_URL=https://api.example.com

// Update imports
// Before
// const apiUrl = import.meta.env.VITE_API_URL;

// After
const apiUrl = process.env.NEXT_PUBLIC_API_URL;
```

## Roadmap and Future Features

### Current Status (2026)

```typescript
// Stable Features
const stableFeatures = [
  'Development mode with HMR',
  'TypeScript/JSX support',
  'CSS Modules and PostCSS',
  'Image optimization',
  'Code splitting',
  'Tree shaking'
];

// Experimental Features
const experimentalFeatures = [
  'Production builds (beta)',
  'Custom loaders API',
  'Standalone CLI (non-Next.js)',
  'Source maps generation',
  'Advanced caching strategies'
];

// Upcoming Features (Roadmap)
const upcomingFeatures = [
  'Full production build parity with Webpack',
  'Plugin ecosystem (public API)',
  'WebAssembly support',
  'Worker thread optimization',
  'Remote caching for CI/CD',
  'Framework-agnostic version'
];
```

## Common Mistakes

### 1. Using Webpack-Specific APIs

```typescript
// Wrong: Using webpack require.context
const images = require.context('./images', false, /\.png$/);

// Correct: Use dynamic imports
const images = import.meta.glob('./images/*.png');

// Or static imports
import image1 from './images/photo1.png';
import image2 from './images/photo2.png';
```

### 2. Incorrect Environment Variable Prefix

```typescript
// Wrong: Server variables exposed to client
// .env.local
API_SECRET=secret-key

// client-component.tsx
const secret = process.env.API_SECRET; // undefined in browser!

// Correct: Use NEXT_PUBLIC_ prefix for client
// .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
API_SECRET=secret-key

// client-component.tsx
const apiUrl = process.env.NEXT_PUBLIC_API_URL; // Works!

// server-component.tsx
const secret = process.env.API_SECRET; // Works on server!
```

### 3. Not Using --turbo Flag

```bash
# Wrong: Still using Webpack
npm run dev

# Correct: Enable Turbopack
npm run dev -- --turbo

# Or update package.json permanently
{
  "scripts": {
    "dev": "next dev --turbo"
  }
}
```

### 4. Incompatible Dependencies

```typescript
// Wrong: Using webpack-only plugins
// next.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer');

module.exports = {
  webpack: (config) => {
    config.plugins.push(new BundleAnalyzerPlugin()); // Won't work with Turbopack
    return config;
  }
};

// Correct: Use Turbopack-compatible alternatives
module.exports = {
  experimental: {
    turbo: {
      // Use built-in features or wait for plugin API
    }
  }
};
```

## Best Practices

### 1. Optimize Bundle Size

```typescript
// Use dynamic imports for large dependencies
const HeavyChart = dynamic(() => import('heavy-chart-library'), {
  loading: () => <Spinner />,
  ssr: false
});

// Use tree-shakable imports
import { Button, Modal } from '@/components'; // Bad: imports everything
import Button from '@/components/Button'; // Good: imports only Button

// Configure package.json for libraries
{
  "sideEffects": ["*.css", "*.scss"], // Mark side-effect-free modules
  "type": "module" // Use ESM for better tree shaking
}
```

### 2. Leverage Caching

```typescript
// Use stable dependency versions for consistent caching
// package.json
{
  "dependencies": {
    "react": "18.2.0", // Exact version, not ^18.2.0
    "next": "14.2.0"
  }
}

// Use Next.js caching strategies
export const revalidate = 3600; // ISR caching

export async function generateStaticParams() {
  // Pre-generate static pages for optimal caching
  return [{ id: '1' }, { id: '2' }];
}
```

### 3. Monitor Build Performance

```typescript
// Create a build metrics script
// scripts/measure-build.ts
const { performance } = require('perf_hooks');

const startTime = performance.now();

process.on('exit', () => {
  const duration = performance.now() - startTime;
  console.log(`Build completed in ${duration.toFixed(2)}ms`);
  
  // Log to analytics
  logBuildMetrics({
    duration,
    timestamp: Date.now(),
    nodeVersion: process.version
  });
});

// Run build
require('next/dist/cli/next-build').nextBuild();
```

### 4. Use Incremental Type Checking

```json
// tsconfig.json
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}
```

## When to Use Turbopack

### Ideal Use Cases

1. **Large Next.js Applications**: Projects with thousands of modules benefit most from incremental bundling
2. **Rapid Development Workflow**: Teams prioritizing fast HMR and iteration speed
3. **Monorepo Projects**: Multiple apps sharing packages with efficient caching
4. **Modern Tech Stack**: TypeScript, React 18+, and modern JavaScript features

### When to Stick with Alternatives

```typescript
// Use Webpack if:
const useWebpack = [
  'Need specific webpack plugins not yet supported',
  'Production builds require absolute stability (Turbopack still maturing)',
  'Complex custom loaders and transformations',
  'Legacy codebase with webpack-specific code'
];

// Use Vite if:
const useVite = [
  'Non-Next.js React/Vue/Svelte project',
  'Prefer unbundled ESM development experience',
  'Established Vite plugin ecosystem needed',
  'Smaller to medium-sized applications'
];

// Use esbuild if:
const useEsbuild = [
  'Building libraries (not full applications)',
  'Simple, fast builds without advanced features',
  'Build tool integration (e.g., as Vite plugin)',
  'Command-line bundling scripts'
];
```

## Interview Questions

### 1. How does Turbopack achieve faster HMR than Webpack?

**Answer:** Turbopack uses an incremental computation model where only changed functions and their direct dependents are recompiled. It maintains a persistent cache of compiled modules and uses a fine-grained dependency graph to determine exactly what needs updating. Built in Rust, it leverages native performance and parallel processing. In contrast, Webpack often rebuilds entire module graphs and has JavaScript-based performance bottlenecks.

### 2. What's the difference between Turbopack's bundling strategy and Vite's unbundled ESM approach?

**Answer:** Vite serves individual ES modules directly to the browser during development (unbundled), relying on native browser ESM support. This is fast for initial loads but can slow down with many modules due to browser request limits. Turbopack bundles modules incrementally in development (similar to production), but only rebundles changed portions. This provides consistent performance at scale and works with all browsers, while Vite requires modern browsers with ESM support.

### 3. Why is Turbopack written in Rust instead of JavaScript?

**Answer:** Rust provides memory safety without garbage collection overhead, enabling predictable performance. It supports fearless concurrency for parallel processing without data races, making it ideal for intensive bundling tasks. Rust's zero-cost abstractions mean high-level code compiles to efficient machine code comparable to C/C++. This results in significantly faster compilation times and lower memory usage compared to JavaScript-based bundlers.

### 4. How does Turbopack handle code splitting and tree shaking?

**Answer:** Turbopack automatically analyzes the module graph to identify dynamic imports and creates separate chunks for code splitting. It performs static analysis to detect unused exports and removes them during the bundling process (tree shaking). The bundler respects the `sideEffects` field in package.json to enable aggressive dead code elimination. Turbopack's incremental architecture means tree shaking and code splitting decisions are cached and only recomputed when dependencies change.

### 5. What are the limitations of Turbopack as of 2026?

**Answer:** While production build support is in beta, it's not yet feature-complete compared to Webpack. The plugin API is still being developed, limiting extensibility. Some webpack-specific plugins and loaders aren't compatible yet. The standalone (non-Next.js) CLI is experimental. Source map generation has some edge cases. However, for Next.js development mode, Turbopack is stable and production-ready.

### 6. How does Turbopack optimize monorepo builds?

**Answer:** Turbopack caches compiled modules across workspace packages, so shared dependencies are only compiled once. It understands workspace protocols and resolves packages efficiently. The incremental bundling means changes in one package only trigger recompilation of affected packages. Combined with tools like Nx or Turborepo, it can leverage distributed caching across CI/CD and developer machines.

### 7. What strategies does Turbopack use for persistent caching?

**Answer:** Turbopack uses content-addressed caching where each module's cache key is derived from its content and dependencies. The cache is persistent across restarts in `.next/cache`. Function-level caching means even within a file, only changed functions are recompiled. The cache is automatically invalidated when dependencies change, ensuring correctness while maximizing reuse.

## Key Takeaways

1. **Incremental bundling** is the core innovation that enables Turbopack's speed, only rebuilding changed modules and their dependents
2. **Rust-based architecture** provides native performance, memory safety, and efficient parallel processing
3. **Function-level caching** is more granular than file-level, maximizing cache reuse
4. **HMR performance scales** consistently regardless of application size, from hundreds to tens of thousands of modules
5. **Zero configuration** for Next.js projects makes adoption seamless with just a `--turbo` flag
6. **Production builds** are still maturing as of 2026, with development mode being fully stable
7. **Tree shaking and code splitting** are automatic and optimized through static analysis
8. **Monorepo support** is first-class with efficient cross-package caching
9. **Migration from Webpack** is straightforward for Next.js projects but requires adapting webpack-specific code
10. **Performance benchmarks** show 10-700x faster HMR than Webpack 5 depending on application size

## Resources

- **Official Documentation**: https://nextjs.org/docs/architecture/turbopack
- **GitHub Repository**: https://github.com/vercel/turbo
- **Performance Benchmarks**: https://turbo.build/pack/docs/benchmarks
- **Next.js Migration Guide**: https://nextjs.org/docs/app/building-your-application/upgrading
- **Vercel Blog**: https://vercel.com/blog/turbopack
- **Rust Book** (for understanding Turbopack internals): https://doc.rust-lang.org/book/
- **Next.js Discord**: Community support and discussions
- **Turbopack Roadmap**: https://github.com/vercel/turbo/projects
- **Bundle Analysis Tools**: https://bundlejs.com/
- **Video Tutorial Series**: Next.js YouTube channel tutorials on Turbopack
