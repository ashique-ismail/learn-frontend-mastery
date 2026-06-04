# Vite and ESBuild

## Overview

Vite is a next-generation frontend build tool that leverages native ES modules and ESBuild for lightning-fast development and optimized production builds. Created by Evan You (Vue.js creator), Vite offers instant server start, blazing-fast Hot Module Replacement (HMR), and optimized builds using Rollup. ESBuild, written in Go, provides extremely fast bundling and transpilation that's 10-100x faster than traditional JavaScript-based bundlers.

## Core Concepts

### Vite Architecture

```
Development Mode:
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │ Native ESM imports
       ▼
┌──────────────┐
│  Vite Server │
├──────────────┤
│  - Route     │
│  - Transform │ ← ESBuild (TypeScript, JSX)
│  - HMR       │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Source Files │
│ (No Bundle)  │
└──────────────┘

Production Mode:
┌──────────────┐
│ Source Files │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   ESBuild    │ Pre-bundle dependencies
│  +  Rollup   │ Bundle application
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Optimized    │
│   Bundle     │
└──────────────┘
```

## Getting Started with Vite

### Create Vite Project

```bash
# Create new project
npm create vite@latest my-app -- --template react-ts
# or
yarn create vite my-app --template react-ts
# or
pnpm create vite my-app --template react-ts

# Available templates:
# vanilla, vanilla-ts
# vue, vue-ts
# react, react-ts
# preact, preact-ts
# lit, lit-ts
# svelte, svelte-ts

cd my-app
npm install
npm run dev
```

### Basic Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  
  // Development server
  server: {
    port: 3000,
    open: true,
    cors: true,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  },
  
  // Build configuration
  build: {
    outDir: 'dist',
    sourcemap: true,
    minify: 'esbuild', // or 'terser'
    target: 'esnext',
    chunkSizeWarningLimit: 500
  },
  
  // Path aliases
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@utils': path.resolve(__dirname, './src/utils')
    }
  }
});
```

## React with Vite

### React Plugin Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      // Enable React Fast Refresh
      fastRefresh: true,
      
      // Babel configuration
      babel: {
        plugins: [
          ['@babel/plugin-proposal-decorators', { legacy: true }],
          ['@babel/plugin-proposal-class-properties', { loose: true }]
        ]
      },
      
      // JSX runtime
      jsxRuntime: 'automatic', // or 'classic'
      
      // Include files for Fast Refresh
      include: '**/*.{jsx,tsx}'
    })
  ],
  
  // ESBuild options
  esbuild: {
    jsxInject: `import React from 'react'`, // Auto-import React
    jsxFactory: 'React.createElement',
    jsxFragment: 'React.Fragment',
    logOverride: { 'this-is-undefined-in-esm': 'silent' }
  }
});
```

### React Component Example

```tsx
// src/App.tsx
import { useState } from 'react';
import { Counter } from '@components/Counter';
import logo from './assets/logo.svg';
import './App.css';

function App() {
  const [count, setCount] = useState(0);

  return (
    <div className="App">
      <img src={logo} className="logo" alt="Logo" />
      <h1>Vite + React</h1>
      
      <Counter count={count} onIncrement={() => setCount(count + 1)} />
      
      <p>
        Edit <code>src/App.tsx</code> and save to test HMR
      </p>
    </div>
  );
}

export default App;

// src/components/Counter.tsx
interface CounterProps {
  count: number;
  onIncrement: () => void;
}

export function Counter({ count, onIncrement }: CounterProps) {
  return (
    <div className="counter">
      <button onClick={onIncrement}>
        Count is: {count}
      </button>
    </div>
  );
}
```

## Environment Variables

```typescript
// .env
VITE_API_URL=https://api.example.com
VITE_API_KEY=secret123
VITE_ENABLE_ANALYTICS=true

// .env.development
VITE_API_URL=http://localhost:8080
VITE_DEBUG=true

// .env.production
VITE_API_URL=https://production-api.example.com
VITE_DEBUG=false

// Usage in code
const apiUrl = import.meta.env.VITE_API_URL;
const apiKey = import.meta.env.VITE_API_KEY;
const isDev = import.meta.env.DEV;
const isProd = import.meta.env.PROD;
const mode = import.meta.env.MODE; // 'development' or 'production'

// Type definitions
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_API_KEY: string;
  readonly VITE_ENABLE_ANALYTICS: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

## Asset Handling

```typescript
// Import assets
import logo from './assets/logo.png'; // Returns URL
import styles from './styles.module.css'; // CSS Modules
import data from './data.json'; // JSON is auto-parsed
import Worker from './worker?worker'; // Web Workers

// Static assets
<img src={logo} alt="Logo" />

// Explicit URL import
import imgUrl from './img.png?url';

// Raw string import
import rawSvg from './icon.svg?raw';

// Import as string (for SVGs)
import iconUrl from './icon.svg?url';

// CSS
import './global.css'; // Global CSS
import classes from './Component.module.css'; // CSS Modules

// SCSS/SASS
import './styles.scss';

// vite.config.ts - Asset configuration
export default defineConfig({
  build: {
    assetsDir: 'assets',
    assetsInlineLimit: 4096, // 4kb - inline smaller assets as base64
    
    rollupOptions: {
      output: {
        assetFileNames: (assetInfo) => {
          const info = assetInfo.name.split('.');
          const ext = info[info.length - 1];
          
          if (/png|jpe?g|svg|gif|tiff|bmp|ico/i.test(ext)) {
            return `images/[name]-[hash][extname]`;
          }
          if (/woff|woff2|eot|ttf|otf/i.test(ext)) {
            return `fonts/[name]-[hash][extname]`;
          }
          
          return `assets/[name]-[hash][extname]`;
        }
      }
    }
  }
});
```

## CSS and Styling

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  css: {
    // CSS Modules configuration
    modules: {
      localsConvention: 'camelCaseOnly', // or 'camelCase', 'dashes'
      scopeBehaviour: 'local',
      generateScopedName: '[name]__[local]___[hash:base64:5]'
    },
    
    // CSS preprocessor options
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`,
        includePaths: ['node_modules']
      },
      less: {
        math: 'always',
        relativeUrls: true,
        javascriptEnabled: true
      }
    },
    
    // PostCSS configuration
    postcss: {
      plugins: [
        require('autoprefixer'),
        require('postcss-preset-env')({
          stage: 3,
          features: {
            'nesting-rules': true
          }
        })
      ]
    }
  }
});

// Component with CSS Modules
import styles from './Button.module.scss';

export function Button({ children }: { children: React.ReactNode }) {
  return (
    <button className={styles.button}>
      {children}
    </button>
  );
}

// Button.module.scss
.button {
  padding: 0.5rem 1rem;
  background-color: #0066cc;
  color: white;
  border: none;
  border-radius: 4px;
  
  &:hover {
    background-color: #0052a3;
  }
}
```

## Plugins and Extensions

### Popular Vite Plugins

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';
import svgr from 'vite-plugin-svgr';
import { VitePWA } from 'vite-plugin-pwa';
import compression from 'vite-plugin-compression';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    
    // TypeScript path mapping
    tsconfigPaths(),
    
    // SVG as React components
    svgr({
      svgrOptions: {
        icon: true
      }
    }),
    
    // Progressive Web App
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#ffffff',
        icons: [
          {
            src: '/icon-192.png',
            sizes: '192x192',
            type: 'image/png'
          },
          {
            src: '/icon-512.png',
            sizes: '512x512',
            type: 'image/png'
          }
        ]
      }
    }),
    
    // Compression
    compression({
      algorithm: 'gzip',
      ext: '.gz'
    }),
    compression({
      algorithm: 'brotliCompress',
      ext: '.br'
    }),
    
    // Bundle visualization
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true
    })
  ]
});
```

### Custom Vite Plugin

```typescript
// vite-plugin-custom.ts
import type { Plugin } from 'vite';

export function customPlugin(): Plugin {
  return {
    name: 'vite-plugin-custom',
    
    // Plugin hooks
    configResolved(config) {
      console.log('Config resolved:', config.mode);
    },
    
    transformIndexHtml(html) {
      return html.replace(
        '<title>',
        '<title>My App - '
      );
    },
    
    transform(code, id) {
      if (id.endsWith('.custom')) {
        // Transform custom file type
        return {
          code: transformCustomFile(code),
          map: null
        };
      }
    },
    
    handleHotUpdate({ file, server }) {
      if (file.endsWith('.custom')) {
        console.log('Custom file updated:', file);
        server.ws.send({
          type: 'custom',
          event: 'file-update',
          data: { file }
        });
      }
    }
  };
}

function transformCustomFile(code: string): string {
  // Custom transformation logic
  return code;
}

// Usage
import { customPlugin } from './vite-plugin-custom';

export default defineConfig({
  plugins: [customPlugin()]
});
```

## Code Splitting and Lazy Loading

```typescript
// Dynamic imports with Vite
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// Route-based code splitting
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// Manual chunks configuration
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Group vendor libraries
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@mui/material', '@emotion/react'],
          'utility-vendor': ['lodash', 'date-fns', 'axios']
        }
      }
    }
  }
});

// Function-based manual chunks
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('react') || id.includes('react-dom')) {
              return 'react-vendor';
            }
            if (id.includes('@mui')) {
              return 'mui-vendor';
            }
            return 'vendor';
          }
        }
      }
    }
  }
});
```

## Build Optimization

```typescript
// vite.config.ts - Production optimization
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  
  build: {
    // Output directory
    outDir: 'dist',
    
    // Generate source maps
    sourcemap: true,
    
    // Minification
    minify: 'esbuild', // or 'terser' for better compression
    
    // Target browsers
    target: 'esnext', // or ['es2020', 'edge88', 'firefox78', 'chrome87', 'safari14']
    
    // Chunk size warning limit
    chunkSizeWarningLimit: 500,
    
    // CSS code splitting
    cssCodeSplit: true,
    
    // Rollup options
    rollupOptions: {
      output: {
        // Naming patterns
        entryFileNames: 'js/[name]-[hash].js',
        chunkFileNames: 'js/[name]-[hash].js',
        assetFileNames: (assetInfo) => {
          if (assetInfo.name?.endsWith('.css')) {
            return 'css/[name]-[hash][extname]';
          }
          return 'assets/[name]-[hash][extname]';
        },
        
        // Manual chunks
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom']
        }
      }
    },
    
    // Terser options (if using terser)
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true
      }
    },
    
    // Report compressed size
    reportCompressedSize: true,
    
    // Write bundle to disk during dev
    write: true
  },
  
  // Dependency optimization
  optimizeDeps: {
    include: ['react', 'react-dom', 'react-router-dom'],
    exclude: ['some-large-dependency']
  }
});
```

## ESBuild Configuration

```typescript
// vite.config.ts - ESBuild options
export default defineConfig({
  esbuild: {
    // JSX transformation
    jsxFactory: 'h',
    jsxFragment: 'Fragment',
    jsxInject: `import React from 'react'`,
    
    // Loader for different file types
    loader: {
      '.js': 'jsx',
      '.ts': 'tsx'
    },
    
    // Include files
    include: /\.(tsx?|jsx?)$/,
    exclude: /node_modules/,
    
    // Drop console and debugger
    drop: ['console', 'debugger'],
    
    // Pure annotations for tree shaking
    pure: ['console.log', 'console.info'],
    
    // Target
    target: 'esnext',
    
    // Platform
    platform: 'browser',
    
    // Define constants
    define: {
      '__VERSION__': '"1.0.0"',
      '__DEV__': 'false'
    },
    
    // Charset
    charset: 'utf8',
    
    // Legal comments
    legalComments: 'none'
  }
});
```

## Development Features

### Hot Module Replacement (HMR)

```typescript
// HMR API
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // Handle module updates
    console.log('Module updated');
  });
  
  // Accept updates for specific dependency
  import.meta.hot.accept('./dep.ts', (newDep) => {
    // Handle dependency update
  });
  
  // Self-accepting module
  import.meta.hot.accept();
  
  // Dispose callback
  import.meta.hot.dispose((data) => {
    // Cleanup before module is replaced
    data.savedState = currentState;
  });
  
  // Shared data between hot updates
  import.meta.hot.data.someValue;
  
  // Invalidate module
  import.meta.hot.invalidate();
  
  // Send custom event
  import.meta.hot.send('custom-event', { data: 'value' });
  
  // Listen for custom events
  import.meta.hot.on('custom-event', (data) => {
    console.log(data);
  });
}

// React component with HMR
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// Preserve state during HMR
if (import.meta.hot) {
  import.meta.hot.accept();
}

export default Counter;
```

### Glob Imports

```typescript
// Import all files matching pattern
const modules = import.meta.glob('./modules/*.ts');
// Returns: { './modules/a.ts': () => import('./modules/a.ts'), ... }

// Eager import (bundled immediately)
const eagerModules = import.meta.glob('./modules/*.ts', { eager: true });
// Returns: { './modules/a.ts': Module }

// Import with custom query
const rawModules = import.meta.glob('./assets/*.svg', {
  as: 'raw',
  eager: true
});

// Import specific exports
const namedExports = import.meta.glob('./components/*.tsx', {
  import: 'default',
  eager: true
});

// Example: Dynamic route loading
const pages = import.meta.glob('./pages/**/*.tsx');

async function loadPage(path: string) {
  const page = pages[`./pages${path}.tsx`];
  if (page) {
    const module = await page();
    return module.default;
  }
}

// Example: Load all icons
const icons = import.meta.glob('./icons/*.svg', {
  as: 'raw',
  eager: true
});

export function getIcon(name: string) {
  return icons[`./icons/${name}.svg`];
}
```

## TypeScript Integration

```typescript
// tsconfig.json for Vite
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": false,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "Node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "types": ["vite/client"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@utils/*": ["./src/utils/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}

// vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_API_KEY: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

// Declare module types
declare module '*.svg' {
  import * as React from 'react';
  export const ReactComponent: React.FunctionComponent<
    React.SVGProps<SVGSVGElement> & { title?: string }
  >;
  const src: string;
  export default src;
}

declare module '*.module.css' {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module '*.module.scss' {
  const classes: { readonly [key: string]: string };
  export default classes;
}
```

## Testing with Vite

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'c8',
      reporter: ['text', 'json', 'html']
    }
  }
});

// src/test/setup.ts
import { expect, afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
import matchers from '@testing-library/jest-dom/matchers';

expect.extend(matchers);

afterEach(() => {
  cleanup();
});

// Example test
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { Counter } from './Counter';

describe('Counter', () => {
  it('renders count', () => {
    render(<Counter count={5} onIncrement={() => {}} />);
    expect(screen.getByText(/count is: 5/i)).toBeInTheDocument();
  });
});
```

## Performance Comparison

```
Build Speed Comparison (Large Project):
┌──────────┬──────────┬─────────────┐
│ Tool     │ Dev Start│ Prod Build  │
├──────────┼──────────┼─────────────┤
│ Webpack  │ 30-60s   │ 120-180s    │
│ Vite     │ <1s      │ 20-40s      │
│ ESBuild  │ <1s      │ 5-10s       │
└──────────┴──────────┴─────────────┘

Bundle Size (Optimized):
- Webpack: Comparable
- Vite: Comparable  
- ESBuild: Slightly larger (less aggressive minification)

HMR Speed:
- Webpack: 1-3s
- Vite: <100ms
- Instant regardless of app size
```

## Common Mistakes

1. **Not configuring base path for deployment**
2. **Using CommonJS instead of ESM**
3. **Not optimizing dependencies**
4. **Ignoring browser compatibility (target)**
5. **Not setting up proper environment variables**
6. **Missing type definitions for imports**
7. **Not utilizing code splitting**
8. **Forgetting to configure proxy for APIs**

## Best Practices

1. Use ESM imports consistently
2. Configure proper TypeScript paths
3. Leverage Vite's fast HMR during development
4. Optimize dependencies with `optimizeDeps`
5. Use lazy loading for routes
6. Configure proper build targets for browsers
7. Enable source maps for debugging
8. Use environment variables properly
9. Monitor bundle size with visualizer
10. Keep Vite and plugins updated

## Key Takeaways

1. Vite offers instant dev server start using native ESM
2. ESBuild provides 10-100x faster builds than traditional tools
3. HMR is nearly instantaneous regardless of app size
4. Production builds use Rollup for optimization
5. No bundling during development improves speed
6. Plugin system is powerful and easy to extend
7. First-class TypeScript support
8. Built-in support for modern frameworks
9. Excellent developer experience
10. Production builds are highly optimized

## Resources

- [Vite Documentation](https://vitejs.dev/)
- [ESBuild Documentation](https://esbuild.github.io/)
- [Awesome Vite](https://github.com/vitejs/awesome-vite)
- [Vite Plugin Development](https://vitejs.dev/guide/api-plugin.html)
- [Vitest Testing Framework](https://vitest.dev/)
- [Rollup Documentation](https://rollupjs.org/)
