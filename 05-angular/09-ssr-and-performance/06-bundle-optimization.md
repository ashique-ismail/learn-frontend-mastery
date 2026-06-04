# Bundle Optimization in Angular

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Bundles](#understanding-bundles)
3. [Lazy Loading Modules and Routes](#lazy-loading-modules-and-routes)
4. [Webpack Optimization](#webpack-optimization)
5. [Tree Shaking](#tree-shaking)
6. [Build Budgets](#build-budgets)
7. [Bundle Analyzer](#bundle-analyzer)
8. [Code Splitting Strategies](#code-splitting-strategies)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)
12. [Key Takeaways](#key-takeaways)
13. [Resources](#resources)

## Introduction

Bundle optimization is crucial for Angular application performance. Smaller bundles mean faster downloads, quicker parsing, and better user experience. This comprehensive guide covers lazy loading, tree shaking, webpack configuration, build budgets, and advanced optimization techniques to minimize your Angular application's bundle size.

## Understanding Bundles

### What Are Bundles?

```typescript
/**
 * Angular Production Build Output:
 * 
 * dist/my-app/browser/
 * ├── main.{hash}.js          - Application code
 * ├── polyfills.{hash}.js     - Browser polyfills
 * ├── runtime.{hash}.js       - Webpack runtime
 * ├── styles.{hash}.css       - Global styles
 * └── *.chunk.js              - Lazy-loaded chunks
 * 
 * Bundle Types:
 * 
 * 1. Main Bundle
 *    - Your application code
 *    - Framework code (Angular)
 *    - Dependencies
 * 
 * 2. Polyfills
 *    - Browser compatibility code
 *    - Only loaded for older browsers
 * 
 * 3. Runtime
 *    - Webpack module loader
 *    - Usually very small
 * 
 * 4. Lazy Chunks
 *    - Route-based code splitting
 *    - Loaded on demand
 * 
 * 5. Vendor Chunk (optional)
 *    - Third-party libraries
 *    - Can be cached separately
 */
```

### Analyzing Bundle Size

```bash
# Build for production
ng build --configuration production

# View bundle sizes
ng build --configuration production --stats-json

# Analyze with webpack-bundle-analyzer
npm install -g webpack-bundle-analyzer
webpack-bundle-analyzer dist/my-app/browser/stats.json
```

## Lazy Loading Modules and Routes

### Basic Route Lazy Loading

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component')
      .then(m => m.HomeComponent)
  },
  {
    path: 'about',
    loadComponent: () => import('./about/about.component')
      .then(m => m.AboutComponent)
  },
  {
    path: 'products',
    loadChildren: () => import('./products/products.routes')
      .then(m => m.PRODUCT_ROUTES)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
    canActivate: [authGuard]
  }
];
```

### Lazy Loading with Providers

```typescript
// feature.routes.ts
import { Routes } from '@angular/router';
import { FeatureService } from './feature.service';

export const FEATURE_ROUTES: Routes = [
  {
    path: '',
    providers: [
      FeatureService // Only loaded with this route
    ],
    children: [
      {
        path: '',
        loadComponent: () => import('./feature-list.component')
          .then(m => m.FeatureListComponent)
      },
      {
        path: ':id',
        loadComponent: () => import('./feature-detail.component')
          .then(m => m.FeatureDetailComponent)
      }
    ]
  }
];
```

### Preloading Strategies

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { 
  provideRouter, 
  withPreloading,
  PreloadAllModules,
  PreloadingStrategy,
  Route
} from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { mergeMap } from 'rxjs/operators';
import { routes } from './app.routes';

// Strategy 1: Preload all modules
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules)
    )
  ]
};

// Strategy 2: Custom preloading
export class CustomPreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Only preload routes with preload: true in data
    if (route.data?.['preload']) {
      console.log('Preloading:', route.path);
      return load();
    }
    return of(null);
  }
}

// Strategy 3: Network-aware preloading
export class NetworkAwarePreloading implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const conn = (navigator as any).connection;
    
    // Don't preload on slow connections
    if (conn && (conn.effectiveType === 'slow-2g' || conn.effectiveType === '2g')) {
      return of(null);
    }
    
    // Preload after delay
    return timer(2000).pipe(mergeMap(() => load()));
  }
}

// Usage
export const routes: Routes = [
  {
    path: 'important',
    data: { preload: true }, // Will be preloaded
    loadComponent: () => import('./important.component')
      .then(m => m.ImportantComponent)
  },
  {
    path: 'optional',
    // Won't be preloaded
    loadComponent: () => import('./optional.component')
      .then(m => m.OptionalComponent)
  }
];
```

## Webpack Optimization

### Production Optimization

```typescript
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "optimization": true,
              "outputHashing": "all",
              "sourceMap": false,
              "namedChunks": false,
              "extractLicenses": true,
              "vendorChunk": false,
              "buildOptimizer": true
            }
          }
        }
      }
    }
  }
}
```

### Custom Webpack Configuration

```typescript
// custom-webpack.config.ts
import { Configuration } from 'webpack';

export default (config: Configuration) => {
  // Add compression
  config.plugins?.push(
    new CompressionPlugin({
      test: /\.(js|css|html|svg)$/,
      threshold: 10240,
      minRatio: 0.8
    })
  );
  
  // Configure splitting
  config.optimization = {
    ...config.optimization,
    runtimeChunk: 'single',
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: Infinity,
      minSize: 0,
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name(module: any) {
            // Create separate chunk for each npm package
            const packageName = module.context.match(
              /[\\/]node_modules[\\/](.*?)([\\/]|$)/
            )[1];
            return `npm.${packageName.replace('@', '')}`;
          }
        }
      }
    }
  };
  
  return config;
};

// angular.json - use custom webpack
{
  "architect": {
    "build": {
      "builder": "@angular-builders/custom-webpack:browser",
      "options": {
        "customWebpackConfig": {
          "path": "./custom-webpack.config.ts"
        }
      }
    }
  }
}
```

## Tree Shaking

### Understanding Tree Shaking

```typescript
/**
 * Tree Shaking:
 * Removes unused code from final bundle
 * 
 * Works with:
 * - ES6 modules (import/export)
 * - Side-effect-free code
 * - Proper package.json sideEffects field
 * 
 * Example:
 */

// library.ts
export function usedFunction() {
  return 'I am used';
}

export function unusedFunction() {
  return 'I am not used'; // This will be tree-shaken
}

// app.component.ts
import { usedFunction } from './library';

@Component({
  template: `{{ message }}`
})
export class AppComponent {
  message = usedFunction();
  // unusedFunction is not imported, so it's removed from bundle
}
```

### Optimizing for Tree Shaking

```typescript
// BAD: Imports entire library
import * as _ from 'lodash';

@Component({
  template: `{{ result }}`
})
export class BadComponent {
  result = _.capitalize('hello'); // Entire lodash in bundle
}

// GOOD: Import specific function
import { capitalize } from 'lodash-es';

@Component({
  template: `{{ result }}`
})
export class GoodComponent {
  result = capitalize('hello'); // Only capitalize function
}

// BETTER: Use native JavaScript
@Component({
  template: `{{ result }}`
})
export class BetterComponent {
  result = 'hello'.charAt(0).toUpperCase() + 'hello'.slice(1);
  // No library needed!
}
```

### Configuring Tree Shaking

```json
// package.json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "dist/index.js",
  "module": "dist/index.esm.js",
  "sideEffects": false,
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.js"
    },
    "./feature": {
      "import": "./dist/feature.esm.js",
      "require": "./dist/feature.js"
    }
  }
}
```

## Build Budgets

### Configuring Budgets

```json
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "1mb"
                },
                {
                  "type": "anyComponentStyle",
                  "maximumWarning": "2kb",
                  "maximumError": "4kb"
                },
                {
                  "type": "bundle",
                  "name": "polyfills",
                  "maximumWarning": "100kb",
                  "maximumError": "150kb"
                },
                {
                  "type": "bundle",
                  "name": "main",
                  "maximumWarning": "400kb",
                  "maximumError": "600kb"
                }
              ]
            }
          }
        }
      }
    }
  }
}
```

### Budget Types

```typescript
/**
 * Budget Types:
 * 
 * 1. initial
 *    - Initial bundle size (before lazy loading)
 *    - Most important for performance
 * 
 * 2. allScript
 *    - Total size of all JavaScript files
 * 
 * 3. all
 *    - Total size of all files (JS + CSS + assets)
 * 
 * 4. anyComponentStyle
 *    - Size of any individual component style
 * 
 * 5. bundle
 *    - Size of specific bundle (main, polyfills, etc.)
 * 
 * 6. any
 *    - Size of any file
 */
```

## Bundle Analyzer

### Using webpack-bundle-analyzer

```bash
# Install
npm install --save-dev webpack-bundle-analyzer

# Generate stats
ng build --stats-json

# Analyze
npx webpack-bundle-analyzer dist/my-app/browser/stats.json
```

### Source Map Explorer

```bash
# Install
npm install --save-dev source-map-explorer

# Build with source maps
ng build --source-map

# Analyze
npx source-map-explorer dist/my-app/browser/*.js
```

### Custom Analysis Script

```typescript
// analyze.ts
import { readFileSync, readdirSync, statSync } from 'fs';
import { join } from 'path';

interface BundleInfo {
  name: string;
  size: number;
  gzipSize?: number;
}

function analyzeBundles(distPath: string): BundleInfo[] {
  const files = readdirSync(distPath);
  const bundles: BundleInfo[] = [];
  
  files.forEach(file => {
    if (file.endsWith('.js')) {
      const filePath = join(distPath, file);
      const stats = statSync(filePath);
      
      bundles.push({
        name: file,
        size: stats.size,
      });
    }
  });
  
  // Sort by size
  bundles.sort((a, b) => b.size - a.size);
  
  // Print report
  console.log('\nBundle Analysis:');
  console.log('================\n');
  
  let totalSize = 0;
  bundles.forEach(bundle => {
    const sizeKB = (bundle.size / 1024).toFixed(2);
    console.log(`${bundle.name}: ${sizeKB} KB`);
    totalSize += bundle.size;
  });
  
  console.log(`\nTotal: ${(totalSize / 1024).toFixed(2)} KB`);
  
  return bundles;
}

// Run
analyzeBundles('dist/my-app/browser');
```

## Code Splitting Strategies

### Component-Level Code Splitting

```typescript
// Dynamic component loading
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-dynamic-loader',
  standalone: true,
  template: `
    <button (click)="loadHeavyComponent()">Load Heavy Component</button>
    <div #container></div>
  `
})
export class DynamicLoaderComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  async loadHeavyComponent() {
    // Component code only loaded when clicked
    const { HeavyComponent } = await import('./heavy.component');
    this.container.createComponent(HeavyComponent);
  }
}
```

### Library Code Splitting

```typescript
// BAD: Import entire library
import { Chart } from 'chart.js';

@Component({
  template: `<canvas #chart></canvas>`
})
export class BadChartComponent implements OnInit {
  ngOnInit() {
    new Chart(/* ... */);
  }
}

// GOOD: Lazy load library
@Component({
  template: `
    <button (click)="showChart()">Show Chart</button>
    <canvas #chart *ngIf="chartLoaded"></canvas>
  `
})
export class GoodChartComponent {
  chartLoaded = false;
  
  async showChart() {
    // Chart.js only loaded when needed
    const { Chart } = await import('chart.js');
    this.chartLoaded = true;
    
    setTimeout(() => {
      new Chart(/* ... */);
    });
  }
}
```

### Vendor Chunk Splitting

```typescript
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "vendorChunk": true, // Separate vendor bundle
              "commonChunk": true, // Common code in separate chunk
              "optimization": {
                "scripts": true,
                "styles": true,
                "fonts": true
              }
            }
          }
        }
      }
    }
  }
}
```

## Common Mistakes

### 1. Not Using Lazy Loading

```typescript
// BAD: Everything in main bundle
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'products', component: ProductsComponent },
  { path: 'admin', component: AdminComponent }
];

// GOOD: Lazy load routes
const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home.component')
      .then(m => m.HomeComponent)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES)
  }
];
```

### 2. Importing Entire Libraries

```typescript
// BAD: 500KB lodash in bundle
import _ from 'lodash';
const result = _.capitalize('hello');

// GOOD: 5KB function
import { capitalize } from 'lodash-es';
const result = capitalize('hello');

// BEST: Native JavaScript
const result = 'hello'.charAt(0).toUpperCase() + 'hello'.slice(1);
```

### 3. No Build Budgets

```typescript
// Always configure budgets to catch regressions
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kb",
      "maximumError": "1mb"
    }
  ]
}
```

### 4. Including Development Code

```typescript
// BAD: Development code in production
if (true) { // Dead code
  console.log('Debug info');
}

// GOOD: Use environment flags
if (!environment.production) {
  console.log('Debug info');
}
```

## Best Practices

### 1. Lazy Load Everything Possible

```typescript
const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard.component')
      .then(m => m.DashboardComponent)
  }
];
```

### 2. Use Standalone Components

```typescript
// Smaller bundles with standalone
@Component({
  selector: 'app-feature',
  standalone: true,
  imports: [CommonModule], // Only import what you need
  template: `...`
})
export class FeatureComponent {}
```

### 3. Optimize Third-Party Libraries

```typescript
// Use ES modules for better tree shaking
import { debounce } from 'lodash-es'; // Good
import debounce from 'lodash/debounce'; // Better
```

### 4. Monitor Bundle Size

```bash
# Add to package.json
{
  "scripts": {
    "build:stats": "ng build --stats-json",
    "analyze": "webpack-bundle-analyzer dist/my-app/browser/stats.json"
  }
}
```

### 5. Use Differential Loading

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022" // Modern browsers
  }
}

// Angular automatically creates ES5 version for older browsers
```

### 6. Optimize Images and Assets

```typescript
// Use appropriate image formats
// WebP for modern browsers
// Optimize with tools like imagemin
// Lazy load images below the fold

@Component({
  template: `
    <img 
      loading="lazy" 
      src="large-image.webp"
      alt="Lazy loaded image"
    >
  `
})
export class ImageComponent {}
```

### 7. Remove Unused Dependencies

```bash
# Audit dependencies
npm ls --depth=0

# Remove unused
npm uninstall unused-package

# Check for duplicates
npm dedupe
```

### 8. Use Preloading Strategies Wisely

```typescript
// Preload important routes, not everything
const routes: Routes = [
  {
    path: 'important',
    data: { preload: true },
    loadComponent: () => import('./important.component')
  },
  {
    path: 'rarely-used',
    // Don't preload
    loadComponent: () => import('./rarely-used.component')
  }
];
```

## Interview Questions

### Q1: What is lazy loading in Angular?

**Answer:** Lazy loading is a technique where modules or components are loaded on-demand rather than at application startup. It reduces initial bundle size and improves initial load time.

### Q2: What is tree shaking?

**Answer:** Tree shaking is the process of removing unused code from the final bundle. It works with ES6 modules and eliminates dead code during the build process.

### Q3: How do build budgets help?

**Answer:** Build budgets set size limits for bundles. The build fails or warns when bundles exceed these limits, preventing accidental bundle size increases.

### Q4: What's the difference between vendorChunk and commonChunk?

**Answer:** vendorChunk separates third-party library code into a vendor bundle. commonChunk extracts code shared across multiple lazy-loaded chunks into a common bundle.

### Q5: How does webpack code splitting work?

**Answer:** Webpack analyzes the module graph and creates separate chunks for lazy-loaded routes, dynamic imports, and configured split points. These chunks are loaded on demand.

### Q6: What tools can analyze bundle size?

**Answer:** webpack-bundle-analyzer, source-map-explorer, and bundle-analyzer. They visualize bundle composition and help identify large dependencies.

### Q7: How do you optimize third-party libraries?

**Answer:** Import only needed functions (tree shaking), use ES module versions, consider alternatives, lazy load when possible, and check for smaller alternatives.

### Q8: What is differential loading?

**Answer:** Differential loading creates two builds: modern ES2015+ for modern browsers and ES5 for older browsers. Browsers automatically download the appropriate version.

## Key Takeaways

1. **Lazy load routes and components** to reduce initial bundle size
2. **Use build budgets** to prevent bundle size regressions
3. **Tree shaking removes unused code** - use ES6 imports
4. **Analyze bundles regularly** with webpack-bundle-analyzer
5. **Import specific functions** not entire libraries
6. **Standalone components** enable better code splitting
7. **Preload strategically** - not everything needs preloading
8. **Monitor bundle size** in CI/CD pipeline
9. **Optimize third-party libraries** - consider alternatives
10. **Use source maps** to understand what's in your bundles

## Resources

### Official Documentation
- [Angular Build Optimization](https://angular.dev/tools/cli/build)
- [Lazy Loading](https://angular.dev/guide/routing/common-router-tasks#lazy-loading)
- [Deployment Guide](https://angular.dev/tools/cli/deployment)

### Tools
- webpack-bundle-analyzer
- source-map-explorer
- bundle-buddy
- webpack Bundle Analyzer

### Articles
- "Angular Bundle Optimization" - Web.dev
- "Reducing Bundle Size" - Angular Blog
- "Tree Shaking in Angular" - Angular University

### Video Tutorials
- "Bundle Optimization Techniques" - ng-conf
- "Performance Best Practices" - Angular Connect
- "Build Optimization" - Angular YouTube
