# Vite - Next Generation Frontend Tooling

## Overview

Vite is a modern build tool that provides an extremely fast development experience through native ES modules and esbuild. Created by Evan You (Vue.js creator), Vite offers instant server start, lightning-fast HMR (Hot Module Replacement), and optimized production builds using Rollup. It supports React, Vue, Svelte, and vanilla JavaScript with minimal configuration.

## Installation and Setup

```bash
# Create new Vite project
npm create vite@latest my-app
cd my-app
npm install

# With specific template
npm create vite@latest my-app -- --template react-ts
npm create vite@latest my-app -- --template vue
npm create vite@latest my-app -- --template svelte

# Start dev server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

## Configuration

### Basic Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    open: true,
    host: true,
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    minify: 'esbuild',
  },
  resolve: {
    alias: {
      '@': '/src',
      '@components': '/src/components',
      '@utils': '/src/utils',
    },
  },
});
```

### Advanced Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [
    react({
      // Fast Refresh options
      fastRefresh: true,
      // Babel plugins
      babel: {
        plugins: ['babel-plugin-styled-components'],
      },
    }),
  ],
  
  server: {
    port: 3000,
    strictPort: true,
    host: '0.0.0.0', // Expose to network
    cors: true,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
    hmr: {
      overlay: true,
      clientPort: 3000,
    },
  },
  
  build: {
    outDir: 'dist',
    assetsDir: 'assets',
    sourcemap: process.env.NODE_ENV !== 'production',
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          ui: ['@mui/material'],
        },
      },
    },
    chunkSizeWarningLimit: 1000,
    cssCodeSplit: true,
  },
  
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
      '@utils': path.resolve(__dirname, './src/utils'),
      '@types': path.resolve(__dirname, './src/types'),
    },
    extensions: ['.mjs', '.js', '.ts', '.jsx', '.tsx', '.json'],
  },
  
  css: {
    modules: {
      localsConvention: 'camelCase',
      scopeBehaviour: 'local',
    },
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`,
      },
    },
    devSourcemap: true,
  },
  
  define: {
    __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
    __DEV__: process.env.NODE_ENV === 'development',
  },
  
  optimizeDeps: {
    include: ['react', 'react-dom'],
    exclude: ['@vite/client', '@vite/env'],
    esbuildOptions: {
      target: 'es2020',
    },
  },
});
```

## Plugins Ecosystem

### Official Plugins

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import vue from '@vitejs/plugin-vue';
import legacy from '@vitejs/plugin-legacy';

export default defineConfig({
  plugins: [
    // React with SWC (faster)
    react({
      jsxRuntime: 'automatic',
      fastRefresh: true,
    }),
    
    // Vue support
    vue(),
    
    // Legacy browser support
    legacy({
      targets: ['defaults', 'not IE 11'],
      additionalLegacyPolyfills: ['regenerator-runtime/runtime'],
    }),
  ],
});
```

### Popular Community Plugins

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';
import compression from 'vite-plugin-compression';
import { VitePWA } from 'vite-plugin-pwa';
import svgr from 'vite-plugin-svgr';

export default defineConfig({
  plugins: [
    react(),
    
    // Bundle analyzer
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
    
    // Gzip compression
    compression({
      algorithm: 'gzip',
      ext: '.gz',
    }),
    
    // PWA support
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'My App',
        short_name: 'App',
        description: 'My awesome app',
        theme_color: '#ffffff',
      },
    }),
    
    // SVG as React component
    svgr(),
  ],
});
```

## Environment Variables

```bash
# .env
VITE_API_URL=http://localhost:5000
VITE_APP_TITLE=My App

# .env.production
VITE_API_URL=https://api.production.com
VITE_APP_TITLE=My App

# .env.development
VITE_API_URL=http://localhost:3000
VITE_APP_TITLE=My App (Dev)
```

```typescript
// Using environment variables
const apiUrl = import.meta.env.VITE_API_URL;
const appTitle = import.meta.env.VITE_APP_TITLE;
const isDev = import.meta.env.DEV;
const isProd = import.meta.env.PROD;
const mode = import.meta.env.MODE;

// Type safety
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

## Asset Handling

```typescript
// Importing assets
import logo from './logo.svg';
import styles from './App.module.css';
import data from './data.json';

// URL imports
import imageUrl from './image.png?url';

// Raw imports
import rawContent from './file.txt?raw';

// Inline imports
import inlineWorker from './worker?worker&inline';

// Worker imports
import Worker from './worker?worker';
const worker = new Worker();

// Dynamic imports
const module = await import('./module.js');

// Glob imports
const modules = import.meta.glob('./modules/*.js');
const eagerModules = import.meta.glob('./modules/*.js', { eager: true });

// Public directory
// Files in /public are served at root
// Access: /logo.png (not /public/logo.png)
```

## Code Splitting

```typescript
// Dynamic imports for code splitting
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

// Manual chunks in config
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('react')) {
              return 'react-vendor';
            }
            if (id.includes('@mui')) {
              return 'mui-vendor';
            }
            return 'vendor';
          }
        },
      },
    },
  },
});
```

## Library Mode

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'MyLib',
      formats: ['es', 'umd', 'cjs'],
      fileName: (format) => `my-lib.${format}.js`,
    },
    rollupOptions: {
      external: ['react', 'react-dom'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM',
        },
      },
    },
  },
});

// package.json
{
  "name": "my-lib",
  "type": "module",
  "files": ["dist"],
  "main": "./dist/my-lib.cjs.js",
  "module": "./dist/my-lib.es.js",
  "exports": {
    ".": {
      "import": "./dist/my-lib.es.js",
      "require": "./dist/my-lib.cjs.js"
    }
  }
}
```

## SSR Support

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  ssr: {
    noExternal: ['react', 'react-dom'],
    target: 'node',
  },
});

// server.js
import express from 'express';
import { createServer as createViteServer } from 'vite';

async function createServer() {
  const app = express();
  
  const vite = await createViteServer({
    server: { middlewareMode: true },
    appType: 'custom',
  });
  
  app.use(vite.middlewares);
  
  app.use('*', async (req, res) => {
    try {
      const url = req.originalUrl;
      
      let template = await vite.transformIndexHtml(url, 
        `<!DOCTYPE html>
        <html>
          <body>
            <div id="app"><!--ssr-outlet--></div>
          </body>
        </html>`
      );
      
      const { render } = await vite.ssrLoadModule('/src/entry-server.jsx');
      const appHtml = await render(url);
      
      const html = template.replace('<!--ssr-outlet-->', appHtml);
      
      res.status(200).set({ 'Content-Type': 'text/html' }).end(html);
    } catch (e) {
      vite.ssrFixStacktrace(e);
      res.status(500).end(e.stack);
    }
  });
  
  app.listen(5173);
}

createServer();
```

## Performance Optimization

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
  
  build: {
    // Tree shaking
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        passes: 2,
      },
    },
    
    // Code splitting
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'router': ['react-router-dom'],
        },
      },
    },
    
    // Chunk size optimization
    chunkSizeWarningLimit: 500,
    
    // CSS code splitting
    cssCodeSplit: true,
  },
  
  // Dependency pre-bundling
  optimizeDeps: {
    include: [
      'react',
      'react-dom',
      'react-router-dom',
    ],
  },
});
```

## Common Patterns

### TypeScript Configuration

```typescript
// vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### Monorepo Setup

```typescript
// apps/web/vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@shared': '../../packages/shared/src',
      '@ui': '../../packages/ui/src',
    },
  },
  server: {
    port: 3000,
  },
});

// packages/ui/vite.config.ts
export default defineConfig({
  build: {
    lib: {
      entry: './src/index.ts',
      name: 'UI',
      fileName: 'index',
    },
  },
});
```

## Common Mistakes

1. **Not using import.meta.env**: Using process.env instead of Vite's import.meta.env.

2. **Missing VITE_ prefix**: Environment variables without VITE_ prefix aren't exposed.

3. **Incorrect public path**: Putting assets in /public when they should be imported.

4. **Not optimizing deps**: Not pre-bundling heavy dependencies with optimizeDeps.

5. **Ignoring code splitting**: Not using dynamic imports for route-level splitting.

6. **Poor proxy configuration**: Incorrectly configured API proxy causing CORS issues.

7. **Not using SSR mode**: Building client-only when SSR is needed.

8. **Missing source maps**: Not enabling sourcemaps for production debugging.

9. **Incorrect alias resolution**: Path aliases not matching TypeScript config.

10. **Not leveraging plugins**: Reinventing functionality that plugins provide.

## Best Practices

1. **Use environment variables**: Prefix with VITE_ and type them properly.

2. **Optimize dependencies**: Configure optimizeDeps for heavy packages.

3. **Implement code splitting**: Use dynamic imports for route-level splitting.

4. **Configure proxy**: Set up API proxy to avoid CORS during development.

5. **Use plugins ecosystem**: Leverage community plugins for common tasks.

6. **Enable source maps**: Configure appropriate source maps for debugging.

7. **Optimize build output**: Use manual chunks and analyze bundle size.

8. **Type everything**: Use TypeScript with proper configuration.

9. **Configure aliases**: Set up path aliases for cleaner imports.

10. **Monitor performance**: Use visualizer plugin to analyze bundle.

## When to Use Vite

**Use Vite when:**
- Starting new projects
- Want fast development experience
- Building modern SPA/SSR apps
- Need instant server start
- Want native ESM support
- Building component libraries
- Want minimal configuration

**Consider alternatives when:**
- Need extensive webpack ecosystem
- Working with legacy codebases
- Require specific webpack features
- Building complex monorepos (consider Turbopack)
- Need IE11 support without polyfills

## Interview Questions

### Q1: How does Vite achieve faster dev server start compared to webpack?

**Answer**: Vite leverages native ES modules in the browser, avoiding bundling during development. It only transforms files on-demand when requested by the browser. Webpack bundles the entire application upfront, which takes longer as the app grows. Vite also uses esbuild (written in Go) for dependency pre-bundling, which is 10-100x faster than JavaScript-based bundlers.

### Q2: What's the difference between Vite's dev and production builds?

**Answer**: Development uses native ESM with on-demand transformation via esbuild for fast HMR. Production uses Rollup for optimized bundling with tree-shaking, minification, and code splitting. This separation provides best of both worlds: speed in development, optimization in production. Dev server doesn't bundle; production creates optimized static assets.

### Q3: How does Vite handle CSS and CSS modules?

**Answer**: Vite has built-in support for CSS, CSS Modules, preprocessors (Sass, Less, Stylus). CSS Modules automatically get scoped class names. Import `styles.module.css` to use. PostCSS is automatically applied. CSS code splitting extracts CSS per chunk. Configuration in `css` option of config file.

### Q4: What are Vite plugins and how do they differ from webpack loaders?

**Answer**: Vite plugins are based on Rollup plugin interface with Vite-specific hooks. They transform modules, customize dev server, and modify build. Unlike webpack loaders (sequential transformers), Vite plugins are more powerful with access to full build process. Many Rollup plugins work directly in Vite.

### Q5: How do you configure API proxying in Vite?

**Answer**: Use server.proxy in config:

```typescript
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:5000',
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api/, ''),
    },
  },
}
```

This proxies /api requests to backend server, solving CORS in development.

### Q6: What's dependency pre-bundling in Vite?

**Answer**: Vite pre-bundles dependencies using esbuild on first run. This converts CommonJS/UMD to ESM and combines many small modules into fewer requests. Pre-bundled deps are cached and only rebuilt when dependencies change. Configured via `optimizeDeps` option. Improves performance by reducing HTTP requests and handling non-ESM packages.

### Q7: How does HMR work in Vite compared to webpack?

**Answer**: Vite's HMR leverages native ESM, updating only the changed module and its immediate dependents without full reload. Webpack's HMR rebuilds affected chunks. Vite's is faster because it doesn't bundle during dev, just transforms individual files. HMR updates are nearly instant regardless of app size. API is similar but Vite's implementation is more efficient.

## Key Takeaways

1. Vite provides instant server start and lightning-fast HMR by leveraging native ES modules during development.

2. Development uses esbuild for dependency pre-bundling and transformation, achieving 10-100x faster performance than JavaScript bundlers.

3. Production builds use Rollup for optimized output with tree-shaking, minification, and code splitting.

4. Environment variables must be prefixed with VITE_ to be exposed to client code via import.meta.env.

5. Plugin ecosystem based on Rollup plugins provides extensive functionality for various use cases.

6. Dependency pre-bundling converts CommonJS/UMD to ESM and combines small modules, cached until dependencies change.

7. Server proxy configuration solves CORS issues during development by forwarding API requests to backend.

8. Asset handling supports URL imports, raw imports, worker imports, and glob imports for flexible asset management.

9. Library mode enables building packages with multiple output formats (ESM, CJS, UMD) for distribution.

10. SSR support provides server-side rendering capabilities with minimal configuration and Vite dev server integration.

## Resources

- [Official Documentation](https://vitejs.dev/)
- [Plugin Directory](https://vitejs.dev/plugins/)
- [GitHub Repository](https://github.com/vitejs/vite)
- [Awesome Vite](https://github.com/vitejs/awesome-vite)
- [Migration Guide](https://vitejs.dev/guide/migration.html)
- [API Reference](https://vitejs.dev/guide/api-plugin.html)
