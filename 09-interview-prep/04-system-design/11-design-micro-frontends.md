# Design Micro-Frontend Architecture

## The Idea

**In plain English:** Micro-frontends is a way of splitting a big website into separate, independent pieces — each piece built, updated, and launched by a different team — so they can all work without getting in each other's way. Think of it like each team owning their own mini website that plugs into one bigger website.

**Real-world analogy:** Imagine a shopping mall where each store is independently owned and operated, but they all share the same building, entrance, and hallways. A new owner can renovate their store without shutting down the whole mall.

- The mall building and entrance = the shell app (handles navigation and shared features)
- Each individual store = a separate micro-frontend (owned and deployed by one team)
- The shared hallways and signs = shared utilities like the design system and authentication

---

## What Problem It Solves

A large frontend monolith with multiple teams becomes hard to deploy independently, slows down CI/CD, and creates cross-team coupling. Micro-frontends let teams own their slice end-to-end.

---

## Decomposition Strategies

### By Route (Most Common)

```
shell.example.com/          → Shell App (navigation, auth)
shell.example.com/catalog/  → Catalog MFE (Team Commerce)
shell.example.com/checkout/ → Checkout MFE (Team Payments)
shell.example.com/account/  → Account MFE (Team Identity)
```

Each route is a separate deployable. Shell handles routing and auth; each team owns their bundle.

### By Feature/Widget (Page Composition)

A single page hosts multiple independent MFEs as widgets. Used in dashboards or portals.

```
Dashboard page:
├── Analytics Widget (Team Data)
├── Task List Widget (Team Productivity)
└── Recent Activity Widget (Team Platform)
```

---

## Implementation: Webpack Module Federation

Module Federation (Webpack 5) is the dominant approach — it allows runtime loading of code from separate deployments.

### Shell (Host) Configuration

```js
// shell/webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        catalog: 'catalog@https://catalog.example.com/remoteEntry.js',
        checkout: 'checkout@https://checkout.example.com/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

### Remote (MFE) Configuration

```js
// catalog/webpack.config.js
new ModuleFederationPlugin({
  name: 'catalog',
  filename: 'remoteEntry.js',
  exposes: {
    './App': './src/App',       // expose root
    './ProductCard': './src/components/ProductCard',  // expose component
  },
  shared: {
    react: { singleton: true, requiredVersion: '^18.0.0' },
  },
})
```

### Shell Loading Remote MFE

```tsx
// Shell app
const CatalogApp = React.lazy(() => import('catalog/App'));

function Router() {
  return (
    <Routes>
      <Route path="/catalog/*" element={
        <Suspense fallback={<Loading />}>
          <ErrorBoundary fallback={<MFEError name="Catalog" />}>
            <CatalogApp />
          </ErrorBoundary>
        </Suspense>
      } />
    </Routes>
  );
}
```

---

## Cross-MFE Communication

### 1. Custom Events (Simplest, Decoupled)

```ts
// Checkout MFE emits
window.dispatchEvent(new CustomEvent('mfe:cart-updated', {
  detail: { itemCount: 3 }
}));

// Shell or another MFE listens
window.addEventListener('mfe:cart-updated', (e) => {
  updateCartBadge(e.detail.itemCount);
});
```

### 2. Shared Store (More Structured)

```ts
// Expose a shared store via Module Federation
// shared-store/webpack.config.js
exposes: { './store': './src/store' }

// Both shell and MFEs import the same singleton
import { store } from 'shared-store/store';
```

### 3. Props/Callbacks (Parent-Child MFEs)

```tsx
// Shell passes auth context to MFE
<CatalogApp
  user={currentUser}
  onAddToCart={handleCartUpdate}
  authToken={token}
/>
```

---

## Shell Responsibilities

```
Shell
├── Authentication & session management
├── Navigation bar and routing
├── Global error boundaries (per MFE)
├── Shared design system/tokens
├── Loading each MFE's remoteEntry.js
└── Cross-MFE communication bus
```

---

## Shared Dependencies

Without coordination, each MFE bundles its own React — 4 MFEs = 4 React copies:

```js
// Solution: shared dependencies in Module Federation
shared: {
  react: { singleton: true },        // only one instance allowed
  'react-dom': { singleton: true },
  '@design-system/components': { singleton: true },
}
```

`singleton: true` ensures only one version loads. `requiredVersion` prevents version mismatches.

---

## Authentication Flow

```
User visits shell.example.com
    ↓
Shell authenticates (cookie or token in memory)
    ↓
Shell passes token to each MFE via:
  a) Props passed to MFE component
  b) Custom event when token refreshes
  c) Shared auth module (Module Federation)
    ↓
Each MFE uses token for its API calls
```

---

## Deployment Independence

```
Team Commerce: Deploy catalog at catalog.example.com/remoteEntry.js
Team Payments: Deploy checkout at checkout.example.com/remoteEntry.js

Shell loads latest remoteEntry.js on each page load → automatic updates
No coordination needed between teams for routine releases
```

Rollback: revert the team's deployment independently.

---

## Interview Discussion Points

**Q: What are the drawbacks of micro-frontends?**
Performance (multiple bundles, latency for remoteEntry.js), complexity (cross-team dependency management, shared dependency versioning), testing (integration testing becomes harder), and operational overhead (more deployments to monitor).

**Q: How do you handle a version mismatch between MFEs and the design system?**
Use semantic versioning and set `requiredVersion` in shared config. Run automated compatibility tests. In extreme cases, maintain two versions during transition with a migration path.

**Q: When should you NOT use micro-frontends?**
Small teams (the coordination overhead exceeds the benefit), early-stage products (architecture premature), when teams can deploy the monolith independently anyway.
