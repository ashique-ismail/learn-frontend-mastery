# Brownfield Migration Strategies

## The Idea

**In plain English:** Brownfield migration means updating or replacing old, messy software that is already running and being used by real people — without shutting it down or starting from scratch. Think of it as renovating a house while people are still living in it, rather than tearing it down and building a new one.

**Real-world analogy:** Imagine a busy restaurant that wants to remodel its kitchen. They can't close for six months while workers gut the whole thing — they'd lose all their customers. So instead, they upgrade one station at a time: first the grill, then the prep area, then the storage. Customers keep eating, waitstaff keeps serving, and the kitchen slowly becomes brand new without anyone noticing.

- The restaurant staying open = the app staying live for real users during migration
- Upgrading one kitchen station at a time = migrating one route or component at a time
- The old kitchen equipment still running while new gear is installed = legacy and new code running side by side in production

---

## Overview

Brownfield development is working within an existing codebase — as opposed to greenfield (starting fresh). Migrating a legacy frontend (jQuery, AngularJS, Backbone, outdated React class components) to a modern stack is a common senior engineering challenge. The goal is continuous delivery throughout: users must never experience downtime, and the team must ship features while migrating.

---

## Core Principle: Never a Big Bang

A "big bang" rewrite — freeze features, rewrite everything, release all at once — almost always fails:

- Takes 6–18 months; business needs change
- Hard to maintain feature parity with a system you're replacing while it evolves
- Risk accumulates silently; bugs surface all at once
- Team morale collapses midway

The alternative: **incremental migration** using strategies that let old and new code coexist and be released independently.

---

## Strategy 1: Strangler Fig Pattern

Named after the strangler fig tree that grows around a host tree and eventually replaces it. New functionality is built in the new system; old functionality migrates piece by piece.

```
Phase 1: Proxy routes traffic
─────────────────────────────
Browser → Proxy → Legacy app (all routes)

Phase 2: New routes in new app
─────────────────────────────
Browser → Proxy → /new-feature → New app
                → /* → Legacy app

Phase 3: Migrate routes progressively
─────────────────────────────────────
Browser → Proxy → /products → New app  ← migrated
                → /checkout → New app  ← migrated
                → /account  → Legacy   ← still pending
                → /*        → Legacy

Phase 4: Legacy is gone
─────────────────────────────
Browser → New app (all routes)
```

**Implementation with a reverse proxy (nginx/CDN):**

```nginx
# nginx.conf — route by path prefix
location /new-feature {
    proxy_pass http://new-app:3000;
}
location /products {
    proxy_pass http://new-app:3000;
}
location / {
    proxy_pass http://legacy-app:8080;
}
```

**Key properties:**
- Both systems run in production simultaneously
- New code ships without waiting for full migration
- Rollback is a config change (flip a route back to legacy)
- Feature parity can be validated route by route

---

## Strategy 2: Micro-Frontend Isolation

Embed new framework components inside the legacy shell using Web Components or iframes, or run them as independent micro-frontends.

### Web Component Bridge

```javascript
// Wrap a React component as a Custom Element
class ReactProductCard extends HTMLElement {
  connectedCallback() {
    const props = {
      productId: this.getAttribute('product-id'),
      onAddToCart: (id) => this.dispatchEvent(
        new CustomEvent('add-to-cart', { detail: id, bubbles: true })
      ),
    };

    const root = ReactDOM.createRoot(this);
    root.render(<ProductCard {...props} />);
    this._root = root;
  }

  disconnectedCallback() {
    this._root?.unmount();
  }
}

customElements.define('react-product-card', ReactProductCard);
```

```html
<!-- Inside legacy jQuery / AngularJS template — no framework knowledge needed -->
<react-product-card product-id="42"></react-product-card>

<script>
  // Legacy code listens to custom events
  document.querySelector('react-product-card')
    .addEventListener('add-to-cart', (e) => {
      legacyCart.add(e.detail);
    });
</script>
```

### Angular Elements (for Angular-specific embedding)

```typescript
import { createCustomElement } from '@angular/elements';

// Wrap an Angular component as a Web Component
const ProductCardEl = createCustomElement(ProductCardComponent, { injector });
customElements.define('angular-product-card', ProductCardEl);
```

---

## Strategy 3: Route-Level Code Splitting (SPA-to-SPA)

When migrating between React versions or between frameworks within an SPA, split at the route level:

```tsx
// React Router: some routes use new stack, some use legacy
import { lazy } from 'react';

const LegacyAccountPage = lazy(() => import('./legacy/AccountPage')); // old class components
const NewProductsPage    = lazy(() => import('./new/ProductsPage'));    // migrated

function App() {
  return (
    <Routes>
      <Route path="/products/*" element={<NewProductsPage />} />
      {/* Legacy routes — wrapped in ErrorBoundary; migrate one by one */}
      <Route path="/account/*" element={
        <Suspense fallback={<Spinner />}>
          <ErrorBoundary fallback={<LegacyFallback />}>
            <LegacyAccountPage />
          </ErrorBoundary>
        </Suspense>
      } />
    </Routes>
  );
}
```

---

## Strategy 4: Component-Level Migration (Bottom-Up)

Migrate leaf components first (buttons, inputs, cards), then composite components, then pages, then the shell. Each migrated component is tested and shipped independently.

```
Leaf          → Composite       → Feature         → Page
─────────────────────────────────────────────────────
Button        → ButtonGroup     → ProductActions   → ProductPage
Input         → SearchBar       → SearchPanel      → SearchPage
Avatar        → UserCard        → UserProfile      → AccountPage
```

**Technique: Adapter components**

When the legacy and new component APIs differ, create thin adapters that translate:

```tsx
// Legacy API: callback props, string-only children
// New API: typed props, composition

// Adapter: presents legacy API, renders new component
function LegacyButtonAdapter({ label, onClick, disabled, variant }) {
  const variantMap = { primary: 'contained', secondary: 'outlined' };
  return (
    <NewButton
      onClick={onClick}
      disabled={disabled}
      variant={variantMap[variant] ?? 'contained'}
    >
      {label}
    </NewButton>
  );
}

// Legacy code uses the adapter unchanged
<LegacyButtonAdapter label="Save" onClick={save} variant="primary" />

// New code uses NewButton directly
<NewButton onClick={save} variant="contained">Save</NewButton>
```

When all consumers are migrated, delete the adapter.

---

## Strategy 5: Feature Flag–Driven Migration

Use feature flags to gate new vs legacy implementation of the same feature:

```typescript
// Feature flag controls which implementation runs
function ProductList({ categoryId }: { categoryId: string }) {
  const flags = useFeatureFlags();

  if (flags.isEnabled('new-product-list-v2')) {
    return <NewProductList categoryId={categoryId} />;  // new implementation
  }
  return <LegacyProductList categoryId={categoryId} />; // old implementation
}

// Gradual rollout:
// Week 1: 5% of users → new implementation
// Week 2: 25% → new
// Week 3: 100% → new (if metrics are good)
// Week 4: Delete legacy code, remove flag
```

This enables A/B testing, canary rollout, and instant rollback (flip the flag) without a deployment.

---

## Managing Shared State During Migration

The hardest part of migration is shared state between old and new components running side by side.

```typescript
// Shared event bus — works across any framework boundary
class MigrationEventBus {
  private emitter = new EventTarget();

  emit<T>(event: string, data: T): void {
    this.emitter.dispatchEvent(
      new CustomEvent(event, { detail: data })
    );
  }

  on<T>(event: string, handler: (data: T) => void): () => void {
    const listener = (e: Event) => handler((e as CustomEvent<T>).detail);
    this.emitter.addEventListener(event, listener);
    return () => this.emitter.removeEventListener(event, listener);
  }
}

export const bus = new MigrationEventBus();

// In legacy code (jQuery)
bus.on('cart:updated', (cart) => {
  $('#cart-count').text(cart.itemCount);
});

// In new React component
bus.emit('cart:updated', { itemCount: 3 });
```

---

## Migration Checklist

**Before starting:**
- [ ] Map all routes, components, and data dependencies
- [ ] Set up feature flag infrastructure
- [ ] Establish visual regression testing baseline (screenshot tests)
- [ ] Define "done" criteria per migration unit
- [ ] Agree on a consistent component API for the new system

**During migration:**
- [ ] Migrate one route/component at a time
- [ ] Write tests for new implementations before migrating
- [ ] Use adapters at boundaries; delete them when the boundary is gone
- [ ] Track bundle size — dual-running frameworks increases it
- [ ] Monitor error rates and performance metrics per route

**After a section is migrated:**
- [ ] Delete the legacy code (don't keep it "just in case" — version control is your backup)
- [ ] Remove the adapter
- [ ] Archive or remove the feature flag

---

## Avoiding Common Pitfalls

| Pitfall | Prevention |
|---|---|
| Running two full framework bundles permanently | Set a deadline; make dual-running a temporary phase |
| Adapter layer becomes permanent | Add a ticket to delete it before the sprint ends |
| No shared design system → visual inconsistency | Migrate design tokens first; both stacks use the same CSS variables |
| Legacy and new state get out of sync | Use a single source of truth (URL, localStorage, or event bus) during co-existence |
| Rewriting features instead of migrating | Scope is "same behavior, new code" — new features wait |

---

## Interview Questions

**Q: What's the strangler fig pattern and why is it preferred over a big bang rewrite?**  
A: The strangler fig incrementally replaces an existing system by routing new functionality to a new implementation while legacy routes stay on the old system. A big bang rewrite requires freezing features for months, accumulates risk invisibly, and often ships with regressions. The strangler fig ships value continuously and allows rollback at any point.

**Q: How do you share state between a legacy jQuery app and a new React component running on the same page?**  
A: Use a framework-agnostic event bus (custom events on `EventTarget` or a simple pub/sub module). Both systems subscribe to the same events and emit state changes through it. Avoid framework-specific state managers (Redux, Zustand) as the bridge — they couple the migration path.

**Q: How do feature flags enable safer migrations?**  
A: A flag lets you deploy both implementations to production and route a percentage of traffic to the new one. If metrics degrade, flip the flag — no deployment needed. Once the new implementation is proven, delete the old code and the flag. This decouples deployment from release and enables instant rollback.
