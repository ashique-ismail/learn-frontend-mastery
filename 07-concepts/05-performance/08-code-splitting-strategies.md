# Code Splitting Strategies

## The Idea

**In plain English:** Code splitting is a technique where you break your website's JavaScript into smaller pieces and only load the piece you actually need right now, instead of forcing the browser to download everything at once. Think of it like only downloading the chapters of a book you are about to read, rather than the entire book upfront.

**Real-world analogy:** Imagine a restaurant that only brings dishes to your table as you order them, rather than placing every item on the menu in front of you the moment you sit down.

- The full menu = all the JavaScript code in your app
- Each dish = a separate chunk of code for a specific page or feature
- Ordering a dish = navigating to a page or clicking a button that triggers loading that chunk
- The kitchen preparing it on demand = the browser fetching and running that chunk only when needed

---

## Overview

Code splitting is the practice of dividing your application's JavaScript bundle into smaller chunks that can be loaded on demand. Instead of sending users a massive bundle with code they may never use, code splitting enables you to load only what's necessary for the current page or feature, dramatically improving initial load time and performance.

## Types of Code Splitting

```
Code Splitting Hierarchy
=========================

┌──────────────────────────────────────────────────┐
│             Application Bundle                    │
│                                                   │
│  ┌────────────────────────────────────────────┐ │
│  │        Route-Based Splitting               │ │
│  │  ┌────────────┐  ┌────────────┐           │ │
│  │  │   Home     │  │   About    │  ...      │ │
│  │  └────────────┘  └────────────┘           │ │
│  └────────────────────────────────────────────┘ │
│                                                   │
│  ┌────────────────────────────────────────────┐ │
│  │      Component-Based Splitting             │ │
│  │  ┌────────────┐  ┌────────────┐           │ │
│  │  │   Modal    │  │  Chart     │  ...      │ │
│  │  └────────────┘  └────────────┘           │ │
│  └────────────────────────────────────────────┘ │
│                                                   │
│  ┌────────────────────────────────────────────┐ │
│  │       Vendor/Library Splitting             │ │
│  │  ┌────────────┐  ┌────────────┐           │ │
│  │  │   React    │  │  Lodash    │  ...      │ │
│  │  └────────────┘  └────────────┘           │ │
│  └────────────────────────────────────────────┘ │
│                                                   │
│  ┌────────────────────────────────────────────┐ │
│  │         Manual/Dynamic Imports             │ │
│  │  ┌────────────┐  ┌────────────┐           │ │
│  │  │  Feature A │  │  Feature B │  ...      │ │
│  │  └────────────┘  └────────────┘           │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

## Route-Based Code Splitting

The most common and effective code splitting strategy divides code by route/page.

### React Implementation

```typescript
// App.tsx - Route-based splitting with React Router
import { Suspense, lazy } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Static imports for critical code
import Header from './components/Header';
import Footer from './components/Footer';
import LoadingSpinner from './components/LoadingSpinner';

// Lazy-loaded route components
const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Products = lazy(() => import('./pages/Products'));
const ProductDetail = lazy(() => import('./pages/ProductDetail'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

// Error boundary for lazy loading errors
class LazyLoadErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Lazy loading error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-container">
          <h2>Failed to load page</h2>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

function App() {
  return (
    <BrowserRouter>
      <div className="app">
        <Header />
        
        <main>
          <LazyLoadErrorBoundary>
            <Suspense fallback={<LoadingSpinner />}>
              <Routes>
                <Route path="/" element={<Home />} />
                <Route path="/about" element={<About />} />
                <Route path="/products" element={<Products />} />
                <Route path="/products/:id" element={<ProductDetail />} />
                <Route path="/dashboard" element={<Dashboard />} />
                <Route path="/settings" element={<Settings />} />
              </Routes>
            </Suspense>
          </LazyLoadErrorBoundary>
        </main>

        <Footer />
      </div>
    </BrowserRouter>
  );
}

export default App;

// Advanced: Preloading routes on hover
import { useEffect, useRef } from 'react';

function NavLink({ to, children, preload }: any) {
  const linkRef = useRef<HTMLAnchorElement>(null);
  const timeoutRef = useRef<number>();

  useEffect(() => {
    const link = linkRef.current;
    if (!link || !preload) return;

    const handleMouseEnter = () => {
      // Start preloading after a short delay
      timeoutRef.current = window.setTimeout(() => {
        preload();
      }, 200);
    };

    const handleMouseLeave = () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };

    link.addEventListener('mouseenter', handleMouseEnter);
    link.addEventListener('mouseleave', handleMouseLeave);

    return () => {
      link.removeEventListener('mouseenter', handleMouseEnter);
      link.removeEventListener('mouseleave', handleMouseLeave);
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, [preload]);

  return (
    <a ref={linkRef} href={to}>
      {children}
    </a>
  );
}

// Usage with preloading
function Navigation() {
  const preloadDashboard = () => import('./pages/Dashboard');
  const preloadSettings = () => import('./pages/Settings');

  return (
    <nav>
      <NavLink to="/dashboard" preload={preloadDashboard}>
        Dashboard
      </NavLink>
      <NavLink to="/settings" preload={preloadSettings}>
        Settings
      </NavLink>
    </nav>
  );
}

// Custom hook for route preloading
function useRoutePreload(importFn: () => Promise<any>) {
  const preloadedRef = useRef(false);

  return useCallback(() => {
    if (!preloadedRef.current) {
      preloadedRef.current = true;
      importFn();
    }
  }, [importFn]);
}

// Usage
function SmartNavLink({ to, importFn, children }: any) {
  const preload = useRoutePreload(importFn);

  return (
    <Link
      to={to}
      onMouseEnter={preload}
      onFocus={preload}
    >
      {children}
    </Link>
  );
}
```

### Angular Implementation

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes, PreloadAllModules } from '@angular/router';

const routes: Routes = [
  {
    path: '',
    loadChildren: () => import('./home/home.module').then(m => m.HomeModule)
  },
  {
    path: 'about',
    loadChildren: () => import('./about/about.module').then(m => m.AboutModule)
  },
  {
    path: 'products',
    loadChildren: () => import('./products/products.module').then(m => m.ProductsModule)
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule),
    data: { preload: true } // Custom preload flag
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preload: false } // Don't preload admin routes
  }
];

@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: PreloadAllModules,
      initialNavigation: 'enabledBlocking'
    })
  ],
  exports: [RouterModule]
})
export class AppRoutingModule {}

// Custom preloading strategy
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preloadedModules: string[] = [];

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Preload if route has preload flag set to true
    if (route.data && route.data['preload']) {
      console.log('Preloading: ' + route.path);
      this.preloadedModules.push(route.path || '');
      
      // Delay preloading to not interfere with initial load
      return timer(2000).pipe(
        mergeMap(() => load())
      );
    }
    
    return of(null);
  }
}

// Network-aware preloading strategy
export class NetworkAwarePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const connection = (navigator as any).connection;
    
    if (route.data && route.data['preload']) {
      // Check network conditions
      if (connection) {
        if (connection.saveData) {
          // User has data saver mode enabled
          console.log('Skipping preload (data saver):', route.path);
          return of(null);
        }
        
        if (connection.effectiveType === '4g') {
          // Fast connection, preload immediately
          console.log('Preloading (4g):', route.path);
          return load();
        } else if (connection.effectiveType === '3g') {
          // Slower connection, delay preload
          console.log('Delayed preload (3g):', route.path);
          return timer(5000).pipe(mergeMap(() => load()));
        }
      }
      
      // Default behavior
      return timer(2000).pipe(mergeMap(() => load()));
    }
    
    return of(null);
  }
}

// app.module.ts
@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: NetworkAwarePreloadingStrategy
    })
  ],
  providers: [
    NetworkAwarePreloadingStrategy
  ]
})
export class AppModule {}

// Lazy loaded module example
// products/products.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ProductsComponent } from './products.component';
import { ProductDetailComponent } from './product-detail/product-detail.component';

@NgModule({
  declarations: [
    ProductsComponent,
    ProductDetailComponent
  ],
  imports: [
    CommonModule,
    RouterModule.forChild([
      { path: '', component: ProductsComponent },
      { path: ':id', component: ProductDetailComponent }
    ])
  ]
})
export class ProductsModule {}
```

## Component-Based Code Splitting

Split code at the component level for heavy or rarely-used components.

```typescript
// React: Component-based splitting
import { Suspense, lazy, useState } from 'react';

// Heavy components loaded on demand
const ChartComponent = lazy(() => import('./components/Chart'));
const VideoPlayer = lazy(() => import('./components/VideoPlayer'));
const RichTextEditor = lazy(() => import('./components/RichTextEditor'));
const DataTable = lazy(() => import('./components/DataTable'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  const [showEditor, setShowEditor] = useState(false);

  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
      
      {/* Load chart only when user requests it */}
      <button onClick={() => setShowChart(true)}>
        Show Chart
      </button>
      
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <ChartComponent data={chartData} />
        </Suspense>
      )}

      {/* Load editor only when user starts editing */}
      {showEditor ? (
        <Suspense fallback={<div>Loading editor...</div>}>
          <RichTextEditor onSave={handleSave} />
        </Suspense>
      ) : (
        <button onClick={() => setShowEditor(true)}>
          Edit Content
        </button>
      )}
    </div>
  );
}

// Modal component with lazy loading
function ModalButton({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  const Modal = lazy(() => import('./components/Modal'));

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      
      {isOpen && (
        <Suspense fallback={null}>
          <Modal onClose={() => setIsOpen(false)}>
            {children}
          </Modal>
        </Suspense>
      )}
    </>
  );
}

// Conditional loading based on feature flags
function FeatureComponent() {
  const { features } = useFeatureFlags();
  
  // Load experimental feature only if enabled
  const ExperimentalFeature = features.newUI
    ? lazy(() => import('./components/ExperimentalFeature'))
    : null;

  return (
    <div>
      {ExperimentalFeature && (
        <Suspense fallback={<div>Loading...</div>}>
          <ExperimentalFeature />
        </Suspense>
      )}
    </div>
  );
}

// Tab-based component splitting
function TabbedInterface() {
  const [activeTab, setActiveTab] = useState('overview');

  const tabs = {
    overview: lazy(() => import('./tabs/OverviewTab')),
    analytics: lazy(() => import('./tabs/AnalyticsTab')),
    settings: lazy(() => import('./tabs/SettingsTab')),
    advanced: lazy(() => import('./tabs/AdvancedTab'))
  };

  const ActiveTabComponent = tabs[activeTab];

  return (
    <div className="tabbed-interface">
      <div className="tab-buttons">
        {Object.keys(tabs).map(tab => (
          <button
            key={tab}
            onClick={() => setActiveTab(tab)}
            className={activeTab === tab ? 'active' : ''}
          >
            {tab}
          </button>
        ))}
      </div>

      <div className="tab-content">
        <Suspense fallback={<div>Loading tab...</div>}>
          <ActiveTabComponent />
        </Suspense>
      </div>
    </div>
  );
}
```

**Angular Implementation:**

```typescript
// Dynamic component loading in Angular
import { Component, ViewChild, ViewContainerRef, ComponentRef } from '@angular/core';

@Component({
  selector: 'app-dynamic-loader',
  template: `
    <button (click)="loadChart()">Load Chart</button>
    <button (click)="loadEditor()">Load Editor</button>
    
    <ng-template #dynamicContainer></ng-template>
  `
})
export class DynamicLoaderComponent {
  @ViewChild('dynamicContainer', { read: ViewContainerRef })
  container!: ViewContainerRef;

  async loadChart() {
    const { ChartComponent } = await import('./chart/chart.component');
    this.container.clear();
    const componentRef = this.container.createComponent(ChartComponent);
    componentRef.instance.data = this.chartData;
  }

  async loadEditor() {
    const { EditorComponent } = await import('./editor/editor.component');
    this.container.clear();
    const componentRef = this.container.createComponent(EditorComponent);
    componentRef.instance.content = this.content;
  }

  chartData = [/* chart data */];
  content = '';
}

// Service for managing dynamic components
import { Injectable, ComponentFactory, ComponentFactoryResolver } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class DynamicComponentService {
  private loadedComponents = new Map<string, any>();

  constructor(private resolver: ComponentFactoryResolver) {}

  async loadComponent(componentPath: string): Promise<any> {
    if (this.loadedComponents.has(componentPath)) {
      return this.loadedComponents.get(componentPath);
    }

    const module = await import(componentPath);
    const component = Object.values(module)[0];
    this.loadedComponents.set(componentPath, component);
    return component;
  }
}
```

## Vendor/Library Splitting

Separate third-party libraries from application code for better caching.

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // Vendor chunk for node_modules
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10,
          reuseExistingChunk: true
        },
        
        // React ecosystem
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router-dom)[\\/]/,
          name: 'react-vendor',
          priority: 20
        },
        
        // Large libraries
        lodash: {
          test: /[\\/]node_modules[\\/]lodash[\\/]/,
          name: 'lodash',
          priority: 20
        },
        
        // UI library
        ui: {
          test: /[\\/]node_modules[\\/](@mui|@material-ui)[\\/]/,
          name: 'ui-vendor',
          priority: 20
        },
        
        // Common code used across routes
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
          name: 'common'
        }
      }
    },
    
    // Runtime chunk for webpack runtime
    runtimeChunk: {
      name: 'runtime'
    }
  }
};

// Vite configuration
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunks
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@mui/material', '@mui/icons-material'],
          'chart-vendor': ['chart.js', 'react-chartjs-2'],
          'utils': ['lodash', 'date-fns', 'axios']
        }
      }
    },
    
    // Chunk size warnings
    chunkSizeWarningLimit: 500
  }
});

// Next.js automatic vendor splitting
// next.config.js
module.exports = {
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.optimization.splitChunks = {
        chunks: 'all',
        cacheGroups: {
          default: false,
          vendors: false,
          
          // Framework chunk (React, Next.js)
          framework: {
            name: 'framework',
            test: /[\\/]node_modules[\\/](react|react-dom|next)[\\/]/,
            priority: 40,
            enforce: true
          },
          
          // Library chunk
          lib: {
            test: /[\\/]node_modules[\\/]/,
            name: 'lib',
            priority: 30,
            minChunks: 1,
            reuseExistingChunk: true
          },
          
          // Common chunk
          commons: {
            name: 'commons',
            minChunks: 2,
            priority: 20
          }
        }
      };
    }
    
    return config;
  }
};
```

## Dynamic Imports for Features

```typescript
// React: Dynamic imports based on user interaction
import { useState } from 'react';

function ImageGallery() {
  const [lightboxOpen, setLightboxOpen] = useState(false);

  const openLightbox = async () => {
    // Load lightbox library only when needed
    const { default: Lightbox } = await import('react-image-lightbox');
    setLightboxOpen(true);
  };

  return (
    <div className="gallery">
      <img src="thumb.jpg" onClick={openLightbox} />
    </div>
  );
}

// Loading PDF viewer on demand
function DocumentViewer({ url }: { url: string }) {
  const [PDFViewer, setPDFViewer] = useState<any>(null);

  useEffect(() => {
    if (url.endsWith('.pdf')) {
      import('react-pdf').then(module => {
        setPDFViewer(() => module.Document);
      });
    }
  }, [url]);

  if (!PDFViewer) {
    return <div>Loading PDF viewer...</div>;
  }

  return <PDFViewer file={url} />;
}

// Loading internationalization on demand
async function loadTranslations(locale: string) {
  const translations = await import(`./locales/${locale}.json`);
  return translations.default;
}

function LanguageSwitcher() {
  const [locale, setLocale] = useState('en');
  const [translations, setTranslations] = useState({});

  const switchLanguage = async (newLocale: string) => {
    const newTranslations = await loadTranslations(newLocale);
    setTranslations(newTranslations);
    setLocale(newLocale);
  };

  return (
    <div>
      <button onClick={() => switchLanguage('en')}>English</button>
      <button onClick={() => switchLanguage('es')}>Español</button>
      <button onClick={() => switchLanguage('fr')}>Français</button>
    </div>
  );
}

// Code splitting utilities
const loadWithRetry = async (
  importFn: () => Promise<any>,
  retries = 3,
  delay = 1000
): Promise<any> => {
  try {
    return await importFn();
  } catch (error) {
    if (retries === 0) throw error;
    
    console.warn(`Import failed, retrying... (${retries} attempts left)`);
    await new Promise(resolve => setTimeout(resolve, delay));
    return loadWithRetry(importFn, retries - 1, delay * 2);
  }
};

// Usage
const LazyComponent = lazy(() =>
  loadWithRetry(() => import('./components/HeavyComponent'))
);

// Preloading helper
const preloadComponent = (importFn: () => Promise<any>) => {
  // Trigger the import but don't use it yet
  importFn();
};

// Preload on hover
function PreloadLink({ to, importFn, children }: any) {
  return (
    <Link
      to={to}
      onMouseEnter={() => preloadComponent(importFn)}
      onFocus={() => preloadComponent(importFn)}
    >
      {children}
    </Link>
  );
}
```

## Bundle Analysis

```typescript
// Analyzing bundle size
import { useState, useEffect } from 'react';

interface BundleStats {
  totalSize: number;
  chunks: Array<{
    name: string;
    size: number;
    modules: string[];
  }>;
}

function BundleAnalyzer() {
  const [stats, setStats] = useState<BundleStats | null>(null);

  useEffect(() => {
    // Load bundle stats (generated by webpack-bundle-analyzer)
    fetch('/bundle-stats.json')
      .then(res => res.json())
      .then(setStats);
  }, []);

  if (!stats) return <div>Loading stats...</div>;

  return (
    <div className="bundle-analyzer">
      <h2>Bundle Analysis</h2>
      <p>Total Size: {(stats.totalSize / 1024).toFixed(2)} KB</p>
      
      <div className="chunks">
        {stats.chunks.map(chunk => (
          <div key={chunk.name} className="chunk">
            <h3>{chunk.name}</h3>
            <p>Size: {(chunk.size / 1024).toFixed(2)} KB</p>
            <details>
              <summary>Modules ({chunk.modules.length})</summary>
              <ul>
                {chunk.modules.map((mod, i) => (
                  <li key={i}>{mod}</li>
                ))}
              </ul>
            </details>
          </div>
        ))}
      </div>
    </div>
  );
}

// webpack-bundle-analyzer configuration
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json',
      statsOptions: { source: false }
    })
  ]
};
```

## Code Splitting Best Practices Flow

```
Code Splitting Decision Tree
=============================

       Start: New Feature/Component
                   │
                   ▼
        ┌──────────────────────┐
        │  Is it critical for  │
        │  initial page load?  │
        └──────────┬───────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
        Yes                 No
         │                   │
         ▼                   ▼
  ┌─────────────┐    ┌──────────────┐
  │ Static      │    │ Lazy Load    │
  │ Import      │    │ Component    │
  └─────────────┘    └──────┬───────┘
                             │
                             ▼
                ┌────────────────────────┐
                │ Will user always need  │
                │ this immediately?      │
                └──────┬─────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
             Yes               No
              │                 │
              ▼                 ▼
    ┌──────────────┐    ┌─────────────┐
    │ Preload on   │    │ Load on     │
    │ hover/focus  │    │ interaction │
    └──────────────┘    └─────────────┘
```

## Common Mistakes

1. **Over-splitting**
   - Problem: Too many small chunks increase HTTP overhead
   - Solution: Find balance, aim for 100-300KB chunks

2. **Not handling loading states**
   - Problem: Poor UX during chunk loading
   - Solution: Always provide meaningful loading indicators

3. **Splitting critical path code**
   - Problem: Delays initial render
   - Solution: Keep above-the-fold code in main bundle

4. **No error handling**
   - Problem: Failed chunk loads break the app
   - Solution: Implement error boundaries and retry logic

5. **Ignoring cache implications**
   - Problem: Poor repeat visit performance
   - Solution: Use content hashes, optimize vendor splits

6. **Not preloading predictable routes**
   - Problem: Unnecessary delays for common navigation
   - Solution: Preload on hover/focus for likely next pages

7. **Splitting shared dependencies**
   - Problem: Duplicate code across chunks
   - Solution: Configure splitChunks properly

8. **Testing only in development**
   - Problem: Production bundles behave differently
   - Solution: Test with production builds

## Best Practices

1. **Start with route-based splitting**: Easiest wins, clearest boundaries
2. **Measure before optimizing**: Use bundle analyzer to identify opportunities
3. **Set size budgets**: Enforce maximum chunk sizes
4. **Implement loading states**: Always show user feedback
5. **Handle errors gracefully**: Provide retry mechanisms
6. **Preload intelligently**: Anticipate user actions
7. **Optimize vendor chunks**: Separate stable libraries
8. **Use content hashing**: Enable long-term caching
9. **Test on slow networks**: Verify loading experience
10. **Monitor in production**: Track chunk load times and errors

## When to Use

**Use code splitting when:**
- Application bundle exceeds 200-300KB
- You have distinct routes/pages
- Heavy components aren't always needed
- Third-party libraries are large
- Initial load time is slow
- Targeting mobile users

**May not be needed for:**
- Very small applications (<100KB)
- Single-page dashboards
- Internal tools with fast networks
- Applications with mostly shared code

## Interview Questions

1. **What is code splitting and why is it important?**
   - Dividing bundle into smaller chunks
   - Reduces initial load time
   - Loads code on demand
   - Improves performance metrics (FCP, LCP, TBT)

2. **Explain the difference between static and dynamic imports.**
   - Static: `import X from 'y'` - loaded at build time, always included
   - Dynamic: `import('y')` - returns Promise, loaded on demand
   - Dynamic enables code splitting

3. **How does React.lazy work internally?**
   - Creates lazy component that loads on render
   - Returns Promise that resolves to component
   - Must be used with Suspense boundary
   - Throws promise during loading (Suspense catches it)

4. **What are the trade-offs of aggressive code splitting?**
   - Pros: Smaller initial bundle, better performance
   - Cons: More HTTP requests, potential waterfalls, complexity
   - Need to balance chunk size vs number of chunks

5. **How would you implement route preloading?**
   - Use link prefetch on hover/focus
   - Call import() without rendering
   - Use Intersection Observer for viewport detection
   - Consider network conditions

6. **Explain webpack's splitChunks optimization.**
   - Automatically splits vendor code
   - Creates common chunks for shared modules
   - Configurable via cacheGroups
   - Prevents duplicate code across chunks

7. **How do you handle code splitting in SSR applications?**
   - Use framework-specific solutions (Next.js dynamic, Loadable Components)
   - Server renders fallback state
   - Client hydrates with actual component
   - Ensure chunk URLs work on server

8. **What strategies would you use for a large component library?**
   - Tree shaking for unused components
   - Separate chunks per component category
   - Lazy load heavy components
   - Document import paths for optimal bundling

## Key Takeaways

1. Code splitting reduces initial bundle size by loading code on demand
2. Route-based splitting provides the biggest wins with least complexity
3. Use React.lazy() and Suspense for component-level splitting
4. Always provide loading states and error boundaries
5. Preload predictable routes on hover/focus for better UX
6. Configure webpack splitChunks to separate vendor code
7. Aim for 100-300KB chunks to balance size vs HTTP overhead
8. Use bundle analyzer to identify splitting opportunities
9. Test with production builds on slow networks
10. Monitor chunk load performance in production

## Resources

- **Official Documentation**
  - [React Code Splitting](https://react.dev/reference/react/lazy)
  - [Angular Lazy Loading](https://angular.io/guide/lazy-loading-ngmodules)
  - [Webpack Code Splitting](https://webpack.js.org/guides/code-splitting/)
  - [Vite Code Splitting](https://vitejs.dev/guide/features.html#code-splitting)

- **Tools**
  - [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
  - [source-map-explorer](https://github.com/danvk/source-map-explorer)
  - [Bundle Buddy](https://bundle-buddy.com/)

- **Articles**
  - [Code Splitting Patterns](https://web.dev/code-splitting-patterns/)
  - [Optimize Long Tasks](https://web.dev/optimize-long-tasks/)
  - [Reduce JavaScript Payloads](https://web.dev/reduce-javascript-payloads-with-code-splitting/)
