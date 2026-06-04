# Module Federation in Production

## Overview

Module Federation (introduced in Webpack 5) enables multiple independently deployed JavaScript applications to share code at runtime — not at build time. A host application can load components, utilities, or entire pages from remote applications that were built and deployed separately, without including that code in its own bundle.

```
Without Module Federation:
  Team A builds checkout → bundles React, shared-utils, checkout components
  Team B builds product-page → bundles React, shared-utils, product components
  Each app: 400KB bundle, duplicated React, duplicated shared code
  Deploying checkout requires rebuilding and deploying the monolith

With Module Federation:
  shared-lib (remote) → deployed at cdn.example.com/shared/remoteEntry.js
  checkout (remote) → deployed at checkout.example.com/remoteEntry.js
  product-page (remote) → deployed at product.example.com/remoteEntry.js
  shell (host) → loads remotes at runtime

  Shell bundle: 50KB — only shell code
  React: loaded ONCE, shared across all remotes
  Deploy checkout independently → shell picks it up without rebuild
```

This is the technical backbone of **Micro Frontend Architecture** — independent teams, independent deployments, independent tech stacks (to a degree), united under a shell host.

## Core Concepts

### Host and Remote

```
Host (Shell):
  - The app that consumes other apps
  - Declares which remotes it loads and their URLs
  - Loads remote code at runtime via async import()

Remote:
  - The app that exposes modules to be consumed
  - Declares which of its modules are exposed
  - Builds a remoteEntry.js manifest

A single app can be both host and remote simultaneously
```

### Shared Dependencies

Module Federation's most powerful and most dangerous feature: shared dependencies allow a single instance of React, lodash, or any package to be used across all federated modules. This prevents multiple React instances (which causes crashes) and reduces total bundle size.

```javascript
// The shared configuration determines:
// 1. Which version is loaded (semver negotiation)
// 2. Whether each app provides its own fallback (singleton vs. eager)
// 3. What happens when versions don't match
```

## webpack.config.js — Host Configuration

```javascript
// shell/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;
const HtmlWebpackPlugin = require('html-webpack-plugin');
const deps = require('./package.json').dependencies;

module.exports = {
  mode: 'production',
  entry: './src/index.ts',
  output: {
    filename: '[name].[contenthash].js',
    publicPath: 'https://shell.example.com/', // CRITICAL: must be absolute URL in production
    clean: true,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        // key: how you import it in code
        // value: remote name @ URL to remoteEntry.js
        checkout: 'checkout@https://checkout.example.com/remoteEntry.js',
        productPage: 'productPage@https://product.example.com/remoteEntry.js',
        // Dynamic remotes via promise (resolved at runtime — see later)
        dynamicRemote: 'dynamicRemote@promise new Promise(resolve => ...)',
      },
      shared: {
        react: {
          singleton: true,      // Only one instance allowed across all remotes
          requiredVersion: deps.react,  // Must match semver range
          eager: false,         // Lazy-load from shared scope, not upfront bundle
        },
        'react-dom': {
          singleton: true,
          requiredVersion: deps['react-dom'],
          eager: false,
        },
        'react-router-dom': {
          singleton: true,
          requiredVersion: deps['react-router-dom'],
          eager: false,
        },
        // Shared utilities
        '@company/design-system': {
          singleton: true,
          requiredVersion: deps['@company/design-system'],
        },
      },
    }),
    new HtmlWebpackPlugin({ template: './public/index.html' }),
  ],
};
```

## webpack.config.js — Remote Configuration

```javascript
// checkout/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;
const deps = require('./package.json').dependencies;

module.exports = {
  mode: 'production',
  entry: './src/index.ts',
  output: {
    filename: '[name].[contenthash].js',
    // MUST be absolute URL — remoteEntry.js references chunks by this path
    publicPath: 'https://checkout.example.com/',
    clean: true,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkout',       // Must match what the host uses: checkout@...
      filename: 'remoteEntry.js',  // The manifest file hosts will load
      exposes: {
        // key: the path used in import('checkout/CheckoutPage')
        // value: the actual file path
        './CheckoutPage': './src/pages/CheckoutPage',
        './CartSummary': './src/components/CartSummary',
        './useCart': './src/hooks/useCart',
      },
      shared: {
        // Must declare same shared deps as host — version negotiation happens here
        react: {
          singleton: true,
          requiredVersion: deps.react,
          eager: false,
        },
        'react-dom': {
          singleton: true,
          requiredVersion: deps['react-dom'],
          eager: false,
        },
      },
    }),
  ],
};
```

## Bootstrap Pattern — Required for Shared Singletons

```typescript
// WRONG — loading a singleton eagerly breaks shared dependency negotiation
// index.ts (direct)
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
ReactDOM.createRoot(document.getElementById('root')!).render(<App />);

// CORRECT — async bootstrap allows Webpack to negotiate shared versions first
// index.ts (entry point — just imports bootstrap)
import('./bootstrap'); // Dynamic import creates an async boundary

// bootstrap.ts — actual app setup
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
ReactDOM.createRoot(document.getElementById('root')!).render(<App />);
```

This pattern is required whenever `eager: false` is used for shared dependencies. Without it, Webpack may load multiple instances of React.

## Consuming Remote Modules in TypeScript

```typescript
// In the host shell — consuming checkout remote
// Types are NOT automatically shared — you need a type declaration

// types/remotes.d.ts
declare module 'checkout/CheckoutPage' {
  import { ComponentType } from 'react';
  const CheckoutPage: ComponentType<{ orderId?: string }>;
  export default CheckoutPage;
}

declare module 'checkout/CartSummary' {
  import { ComponentType } from 'react';
  interface CartSummaryProps { compact?: boolean; }
  const CartSummary: ComponentType<CartSummaryProps>;
  export default CartSummary;
}

declare module 'checkout/useCart' {
  interface CartItem { id: string; qty: number; price: number; }
  interface UseCartResult {
    items: CartItem[];
    total: number;
    addItem: (id: string) => void;
    removeItem: (id: string) => void;
  }
  export function useCart(): UseCartResult;
}
```

```typescript
// shell/src/pages/CartPage.tsx
import React, { Suspense, lazy } from 'react';

// Lazy import from remote — downloaded from checkout.example.com at runtime
const CheckoutPage = lazy(() => import('checkout/CheckoutPage'));
const CartSummary = lazy(() => import('checkout/CartSummary'));

export function CartPage() {
  return (
    <div className="cart-page">
      <Suspense fallback={<CheckoutSkeleton />}>
        <CartSummary compact={false} />
      </Suspense>

      <Suspense fallback={<div>Loading checkout...</div>}>
        <CheckoutPage />
      </Suspense>
    </div>
  );
}

function CheckoutSkeleton() {
  return <div className="skeleton" aria-label="Loading cart..." />;
}
```

## Dynamic Remote Loading

Static remotes are declared in webpack config. Dynamic remotes are resolved at runtime — useful for A/B testing, feature flags, or loading different versions per environment:

```typescript
// Load a remote dynamically — URL determined at runtime
async function loadRemoteModule(
  remoteName: string,
  remoteUrl: string,
  modulePath: string
): Promise<unknown> {
  // Avoid loading the same remote twice
  if (!(window as any)[remoteName]) {
    await new Promise<void>((resolve, reject) => {
      const script = document.createElement('script');
      script.src = remoteUrl;
      script.type = 'text/javascript';
      script.async = true;
      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Failed to load remote: ${remoteUrl}`));
      document.head.appendChild(script);
    });
  }

  // Initialize the remote container
  const container = (window as any)[remoteName];
  await container.init(__webpack_share_scopes__.default);

  // Load the specific module
  const factory = await container.get(modulePath);
  return factory();
}

// Usage in a React component
const RemoteComponent = React.lazy(async () => {
  const remoteUrl = await getFeatureFlagRemoteUrl('checkout');
  const module = await loadRemoteModule('checkout', remoteUrl, './CheckoutPage') as any;
  return { default: module.default };
});
```

## Webpack Config for Dynamic Remotes

```javascript
// Alternative: use promise syntax in webpack config for dynamic URLs
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    checkout: `promise new Promise(resolve => {
      // URL comes from server config, feature flags, etc.
      const remoteUrl = window.__REMOTE_URLS__?.checkout
        ?? 'https://checkout.example.com/remoteEntry.js';
      const script = document.createElement('script');
      script.src = remoteUrl;
      script.onload = () => {
        const proxy = {
          get: (request) => window.checkout.get(request),
          init: (arg) => {
            try {
              return window.checkout.init(arg);
            } catch(e) {
              // ignore already-initialized errors
            }
          }
        };
        resolve(proxy);
      };
      script.onerror = () => resolve({ get: () => null, init: () => {} });
      document.head.appendChild(script);
    })`,
  },
});
```

## Version Negotiation and Conflict Resolution

```
Scenario: Host requires React ^18.0.0, Remote requires React ^17.0.0

Default behavior (singleton: true, strictVersion: false):
  Webpack picks the HIGHER compatible version
  Logs a console warning about version mismatch
  App may work, may have subtle bugs

With strictVersion: true:
  Webpack throws an error and refuses to load the remote
  Safer — prevents subtle compatibility bugs

With singleton: false:
  Each remote loads its own React
  Works correctly but two React instances → Context doesn't cross boundaries
  useContext from host won't see provider in remote and vice versa

Best practice for production:
  1. Treat shared singleton versions as a contract between teams
  2. Use exact versions (not ranges) for singletons: requiredVersion: '18.2.0'
  3. Have a shared platform team own dependency upgrades
  4. Gate major version bumps behind feature flags
```

## Error Boundaries for Remote Failures

Remote modules can fail to load (network error, deployment issue). Always wrap in error boundaries:

```typescript
// RemoteErrorBoundary.tsx
import React, { Component, PropsWithChildren } from 'react';

interface State {
  hasError: boolean;
  error: Error | null;
}

export class RemoteErrorBoundary extends Component<
  PropsWithChildren<{ fallback: React.ReactNode; remoteName: string }>,
  State
> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Track remote loading failures
    analytics.track('remote_load_failed', {
      remote: this.props.remoteName,
      error: error.message,
      stack: info.componentStack,
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// Usage
function CartPage() {
  return (
    <RemoteErrorBoundary
      remoteName="checkout"
      fallback={<FallbackCheckout />}
    >
      <Suspense fallback={<CheckoutSkeleton />}>
        <CheckoutPage />
      </Suspense>
    </RemoteErrorBoundary>
  );
}

function FallbackCheckout() {
  return (
    <div className="checkout-fallback">
      <p>Checkout is temporarily unavailable.</p>
      <a href="https://checkout.example.com">Open checkout directly</a>
    </div>
  );
}
```

## Deployment Strategies

### Independent Deployment

```
Each microfrontend has its own CI/CD pipeline:

checkout team workflow:
  git push → CI runs tests → Build checkout artifacts
  → Deploy to checkout.example.com/remoteEntry.js + chunks
  → Shell picks up new version on NEXT PAGE LOAD (no shell redeploy needed)

Shell NEVER needs to redeploy when a remote updates.
This is the core value proposition of Module Federation.
```

### Versioned Remote URLs

```javascript
// Problem: when shell loads "remoteEntry.js", it may get stale cached version
// Solution: version the remoteEntry URL

// Option 1: Semantic version in URL
// checkout@https://checkout.example.com/v2.3.1/remoteEntry.js

// Option 2: Deployment ID / git SHA
// checkout@https://checkout.example.com/deploys/abc123/remoteEntry.js

// Option 3: Service worker or CDN cache-busting headers
// Set Cache-Control: no-cache on remoteEntry.js
// Set Cache-Control: max-age=31536000 on hashed chunk files

// In production, remoteEntry.js should NOT be cached (it's the manifest)
// The chunks IT references SHOULD be cached (they're content-hashed)
```

### Blue/Green Deployments for Remotes

```typescript
// Config service returns which remote version to load
const remoteConfig = await fetch('/api/remote-config').then(r => r.json());
// { checkout: 'https://checkout-blue.example.com/remoteEntry.js' }

// After testing, switch to green:
// { checkout: 'https://checkout-green.example.com/remoteEntry.js' }

// Shell uses dynamic remotes — no rebuild needed to switch
```

## Shared State Between Remotes

Remotes should be loosely coupled — avoid sharing state directly. Use these patterns:

```typescript
// Pattern 1: Custom Events (simplest, decoupled)
// checkout remote fires an event when cart updates
window.dispatchEvent(new CustomEvent('cart:updated', {
  detail: { itemCount: 3, total: 89.99 }
}));

// Shell or nav remote listens
window.addEventListener('cart:updated', (e: Event) => {
  const { itemCount } = (e as CustomEvent).detail;
  setNavCartCount(itemCount);
});

// Pattern 2: Shared service via Module Federation
// shared-services remote exposes a cart service
// Both shell and checkout import from 'sharedServices/cartService'
// Since it's a singleton, they share the same instance

// Pattern 3: URL as state (most resilient)
// Remotes communicate through URL params and history API
// Works across hard refreshes, doesn't require shared JS runtime
```

## Common Pitfalls

### 1. publicPath Must Be Absolute in Production

```javascript
// WRONG — relative publicPath breaks cross-origin chunk loading
output: { publicPath: '/' }
// Chunks referenced in remoteEntry.js resolve relative to HOST, not remote origin

// CORRECT
output: { publicPath: 'https://checkout.example.com/' }
// Or use 'auto' for same-origin deployments
output: { publicPath: 'auto' }
```

### 2. Missing Async Bootstrap

```typescript
// WRONG — synchronous top-level import prevents shared dep negotiation
import App from './App'; // Eager — breaks shared React
ReactDOM.render(<App />, document.getElementById('root'));

// CORRECT — async boundary allows module federation to negotiate
import('./bootstrap'); // index.ts
// bootstrap.ts contains the actual render
```

### 3. Context Not Crossing Remote Boundaries with Multiple React Instances

```typescript
// Symptom: useContext() returns undefined inside remote component
// Even though Provider wraps it in host shell

// Cause: remote loaded its own React (not shared singleton)
// Provider instance in host-React ≠ Consumer looking in remote-React

// Fix: ensure singleton: true and eager: false for react/react-dom
// Both host and remote must declare the SAME shared config
```

### 4. TypeScript Declaration Maintenance

```typescript
// Remote teams must publish types for their exposed modules
// Options:
// 1. Publish @company/checkout-types npm package (formal, versioned)
// 2. Use @module-federation/typescript plugin (auto-generates and serves types)
// 3. Shared types monorepo (works well in Nx/Turborepo setups)

// @module-federation/typescript setup:
// Remote: plugin downloads and exposes type declarations
// Host: plugin fetches type declarations from remote at build time
```

## Vite + Module Federation

```typescript
// vite.config.ts — using @originjs/vite-plugin-federation
import { defineConfig } from 'vite';
import federation from '@originjs/vite-plugin-federation';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: 'checkout',
      filename: 'remoteEntry.js',
      exposes: {
        './CheckoutPage': './src/pages/CheckoutPage',
      },
      shared: ['react', 'react-dom'],
    }),
  ],
  build: {
    target: 'esnext',
    minify: false,         // Required for @originjs/vite-plugin-federation
    cssCodeSplit: false,   // Prevents CSS loading issues with remotes
  },
});
```

Note: Vite's Module Federation support is less mature than Webpack's. For production use, Webpack remains the more battle-tested choice as of 2025.

## Interview Questions

1. **What problem does Module Federation solve, and how is it different from npm packages?**
   npm packages are shared at build time — every app that uses `@company/button` bundles its own copy at the version they installed. Module Federation shares code at runtime — the host downloads a module from a remote URL when needed, and the remote was built and deployed independently. This enables: (1) independent deployment without rebuilding consumers, (2) a single instance of shared dependencies (like React) across the entire micro-frontend ecosystem, (3) lazy loading of entire application sections, and (4) different teams shipping at their own cadence.

2. **Explain the bootstrap pattern and why it's required for shared singleton dependencies.**
   When a Webpack bundle starts executing, it must decide which version of shared dependencies (like React) to use. This version negotiation requires all federated modules to register in the shared scope before any code using those dependencies runs. If you import React at the top level of `index.ts`, it executes before negotiation completes. The bootstrap pattern wraps the actual app entry in a dynamic `import('./bootstrap')`, creating an async boundary. Webpack's module federation runtime runs the negotiation inside this boundary before executing bootstrap code.

3. **How does version negotiation work for shared dependencies, and what is `singleton: true`?**
   When multiple federated modules declare the same shared package, Webpack compares their `requiredVersion` ranges. For non-singletons, it loads the highest compatible version for each consumer. For `singleton: true`, only ONE version is loaded for the entire page — if versions are incompatible, Webpack either logs a warning and uses the higher version (default) or throws an error (with `strictVersion: true`). Singletons are critical for React, because multiple React instances on the same page cause crashes — React's Context API depends on object identity.

4. **How would you handle a remote module failing to load in production?**
   Wrap every remote module import in both React.Suspense (for loading state) and an ErrorBoundary (for network failures, deployment errors, chunk loading errors). The ErrorBoundary should: (1) render a meaningful fallback UI, (2) log the error to your observability platform with the remote name for debugging, (3) optionally attempt a retry with exponential backoff, (4) optionally provide a direct link to the remote app as a fallback. Never let a remote failure crash the entire shell application.

5. **What is `publicPath` in Module Federation, and what happens if it's wrong?**
   `publicPath` tells Webpack the base URL where all built chunks are hosted. When the shell loads `remoteEntry.js` from `checkout.example.com`, the entry manifest contains relative references to chunk files. Webpack resolves those references against `publicPath` when loading them dynamically. If `publicPath` is '/', Webpack looks for chunks at `shell.example.com/checkout-chunk.js` instead of `checkout.example.com/checkout-chunk.js`, causing 404s. In production, `publicPath` must be the absolute URL of the remote's deployment origin.

6. **How do you share state between independently deployed micro-frontends?**
   Options in order of coupling: (1) Custom DOM Events — most decoupled, works across iframes, no shared runtime required. (2) URL/History API — state in the URL is resilient to page refreshes. (3) Module Federation shared service — expose a singleton service (event bus, global store) from a shared-services remote; since it's a singleton in the shared scope, all remotes get the same instance. (4) BroadcastChannel — cross-tab state without a shared runtime. Avoid sharing React state (Context, Redux store) directly across remotes unless they're guaranteed to share the same React singleton — Context identity breaks across separate React instances.

7. **Describe a versioning strategy for remote deployments that avoids breaking the shell.**
   Best practice: (1) The `remoteEntry.js` manifest is not content-hashed — it's always at the same URL (`/remoteEntry.js`), but should have `Cache-Control: no-cache` so the shell always fetches fresh. (2) All chunk files referenced by `remoteEntry.js` ARE content-hashed (by filename) and set `Cache-Control: max-age=31536000`. (3) Old chunk files are never deleted from the CDN until confirmed no active users have them cached. (4) For blue/green or versioned deployments, use dynamic remote URLs from a config service rather than static URLs in webpack config.

8. **What are the trade-offs of Module Federation versus a monorepo with shared packages?**
   Module Federation: independent deployment (remotes ship without host rebuild), truly independent teams, smaller initial bundle per app, can use different framework versions per remote (with careful sharing config). Drawbacks: runtime complexity (version negotiation, error boundaries, async loading), harder debugging (source maps across domains), operational overhead (each remote is a deployed service), TypeScript types require explicit sharing. Monorepo with shared packages: simpler tooling, type safety is automatic via project references, easier to refactor across boundaries, atomic deployments. Drawback: deploying one team's change requires rebuilding/redeploying the whole product. Choose Module Federation when independent deployment cadences are the primary driver.
