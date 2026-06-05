# Micro Frontends

## The Idea

**In plain English:** Micro frontends are a way of building a website by splitting it into separate mini-apps, each built and updated by a different team, that all appear as one seamless site to the visitor. Think of each mini-app as its own independent piece that can be changed without touching the others.

**Real-world analogy:** Imagine a large shopping mall where different store owners each run their own shop — the food court, the clothing store, and the electronics store are all separate businesses. The mall building (shell) just provides the entrance, hallways, and signs so visitors can move between them:

- The mall building = the shell/container application that loads and displays each section
- Each individual store = a micro frontend owned and deployed by a separate team
- The shared corridors and signage = shared navigation, design system, and routing between micro frontends

---

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Implementation Approaches](#implementation-approaches)
4. [Module Federation](#module-federation)
5. [Implementation Examples](#implementation-examples)
6. [Common Mistakes](#common-mistakes)
7. [Best Practices](#best-practices)
8. [When to Use/Not to Use](#when-to-usenot-to-use)
9. [Interview Questions](#interview-questions)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## Introduction

Micro Frontends extend the microservices concept to frontend development. Instead of building a single monolithic frontend application, you create multiple smaller, independently deployable frontend applications that work together to form a cohesive user experience.

Each micro frontend is owned by a different team, has its own codebase, build process, and deployment pipeline, enabling teams to work independently while delivering a unified application to users.

## Core Concepts

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│            User's Browser                            │
│  ┌───────────────────────────────────────────────┐  │
│  │          Shell/Container Application          │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────────────┐│  │
│  │  │  Nav    │ │ Product │ │   Checkout      ││  │
│  │  │  MFE    │ │   MFE   │ │     MFE         ││  │
│  │  │(Team A) │ │(Team B) │ │   (Team C)      ││  │
│  │  └─────────┘ └─────────┘ └─────────────────┘│  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
        │           │              │
        ▼           ▼              ▼
   ┌────────┐  ┌────────┐    ┌─────────┐
   │ CDN/   │  │ CDN/   │    │  CDN/   │
   │Server A│  │Server B│    │ Server C│
   └────────┘  └────────┘    └─────────┘
```

### Benefits

1. **Independent Deployment**: Deploy micro frontends separately
2. **Team Autonomy**: Teams own features end-to-end
3. **Technology Diversity**: Use different frameworks per micro frontend
4. **Scalability**: Scale teams and code independently
5. **Fault Isolation**: Issues in one MFE don't break others

### Challenges

1. **Complexity**: Additional coordination needed
2. **Performance**: Bundle duplication, loading multiple apps
3. **Consistency**: Maintaining unified UX
4. **Shared State**: Managing state across MFEs
5. **Testing**: Integration testing is harder

## Implementation Approaches

### 1. Build-Time Integration

Packages published to npm and integrated at build time:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Product MFE │     │ Checkout MFE │     │    Nav MFE   │
│              │     │              │     │              │
│ npm publish  │     │ npm publish  │     │ npm publish  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                     │
       └────────────────────┼─────────────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  Shell App      │
                  │  npm install    │
                  │  Build & Deploy │
                  └─────────────────┘
```

```json
// Shell app package.json
{
  "dependencies": {
    "@company/product-mfe": "^1.0.0",
    "@company/checkout-mfe": "^2.1.0",
    "@company/nav-mfe": "^1.5.0"
  }
}
```

```typescript
// Shell app
import { ProductApp } from '@company/product-mfe';
import { CheckoutApp } from '@company/checkout-mfe';
import { Navigation } from '@company/nav-mfe';

function App() {
  return (
    <div>
      <Navigation />
      <Routes>
        <Route path="/products/*" element={<ProductApp />} />
        <Route path="/checkout/*" element={<CheckoutApp />} />
      </Routes>
    </div>
  );
}
```

### 2. Run-Time Integration (iframe)

Simplest but most isolated approach:

```typescript
// Shell app
function App() {
  return (
    <div className="app">
      <Navigation />
      <div className="content">
        <iframe
          src="https://products.example.com"
          title="Products"
          style={{ width: '100%', height: '600px', border: 'none' }}
        />
      </div>
    </div>
  );
}
```

Pros:
- Complete isolation
- Different technologies easily
- Independent deployment

Cons:
- Poor UX (scrolling, routing)
- Performance overhead
- Communication complexity

### 3. Run-Time Integration (JavaScript)

Load micro frontends dynamically at runtime:

```typescript
// Shell app
function loadMicroFrontend(name: string, host: string) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = `${host}/remoteEntry.js`;
    script.onload = () => {
      // @ts-ignore
      resolve(window[name]);
    };
    script.onerror = reject;
    document.head.appendChild(script);
  });
}

// Usage
async function ProductPage() {
  const [ProductMFE, setProductMFE] = useState(null);

  useEffect(() => {
    loadMicroFrontend('ProductMFE', 'https://products.example.com')
      .then(mfe => setProductMFE(() => mfe.App));
  }, []);

  if (!ProductMFE) return <div>Loading...</div>;
  return <ProductMFE />;
}
```

### 4. Web Components

Framework-agnostic using Web Components:

```typescript
// Product MFE (can be any framework)
class ProductWidget extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <div class="product-widget">
        <h2>Products</h2>
        <div id="product-list"></div>
      </div>
    `;
    this.loadProducts();
  }

  loadProducts() {
    // Fetch and render products
  }
}

customElements.define('product-widget', ProductWidget);

// Shell app (framework-agnostic)
function App() {
  return (
    <div>
      <product-widget></product-widget>
      <checkout-widget></checkout-widget>
    </div>
  );
}
```

## Module Federation

Webpack Module Federation enables true runtime integration with shared dependencies:

### Configuration

```javascript
// Product MFE webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'productMFE',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductApp': './src/App',
        './ProductList': './src/components/ProductList'
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' }
      }
    })
  ]
};

// Checkout MFE webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkoutMFE',
      filename: 'remoteEntry.js',
      exposes: {
        './CheckoutApp': './src/App',
        './Cart': './src/components/Cart'
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' }
      }
    })
  ]
};

// Shell app webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        productMFE: 'productMFE@https://products.example.com/remoteEntry.js',
        checkoutMFE: 'checkoutMFE@https://checkout.example.com/remoteEntry.js'
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' }
      }
    })
  ]
};
```

## Implementation Examples

### Example 1: Module Federation with React

```typescript
// ============================================
// Product MFE
// ============================================

// src/App.tsx
import { useState, useEffect } from 'react';
import { ProductList } from './components/ProductList';

export function ProductApp() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(setProducts);
  }, []);

  return (
    <div className="product-app">
      <h1>Products</h1>
      <ProductList products={products} />
    </div>
  );
}

// src/components/ProductList.tsx
interface Product {
  id: string;
  name: string;
  price: number;
}

export function ProductList({ products }: { products: Product[] }) {
  return (
    <div className="product-list">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// src/bootstrap.tsx
import ReactDOM from 'react-dom/client';
import { ProductApp } from './App';

const mount = (el: HTMLElement) => {
  const root = ReactDOM.createRoot(el);
  root.render(<ProductApp />);
};

// Mount if standalone
if (process.env.NODE_ENV === 'development') {
  const el = document.getElementById('root');
  if (el) mount(el);
}

export { mount, ProductApp };

// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  entry: './src/index.ts',
  mode: 'production',
  output: {
    publicPath: 'https://products.example.com/',
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'productMFE',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductApp': './src/App',
        './ProductList': './src/components/ProductList'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};

// ============================================
// Checkout MFE
// ============================================

// src/App.tsx
import { useState } from 'react';
import { Cart } from './components/Cart';
import { CheckoutForm } from './components/CheckoutForm';

export function CheckoutApp() {
  const [step, setStep] = useState<'cart' | 'checkout'>('cart');

  return (
    <div className="checkout-app">
      <h1>Checkout</h1>
      {step === 'cart' ? (
        <Cart onCheckout={() => setStep('checkout')} />
      ) : (
        <CheckoutForm onBack={() => setStep('cart')} />
      )}
    </div>
  );
}

// src/components/Cart.tsx
export function Cart({ onCheckout }: { onCheckout: () => void }) {
  // Use shared cart store
  const { items, total } = useCartStore();

  return (
    <div className="cart">
      {items.map(item => (
        <CartItem key={item.id} item={item} />
      ))}
      <div className="total">Total: ${total}</div>
      <button onClick={onCheckout}>Proceed to Checkout</button>
    </div>
  );
}

// webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkoutMFE',
      filename: 'remoteEntry.js',
      exposes: {
        './CheckoutApp': './src/App',
        './Cart': './src/components/Cart'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};

// ============================================
// Shell App
// ============================================

// src/App.tsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Navigation } from './components/Navigation';
import { ErrorBoundary } from './components/ErrorBoundary';

// Lazy load micro frontends
const ProductApp = lazy(() => import('productMFE/ProductApp'));
const CheckoutApp = lazy(() => import('checkoutMFE/CheckoutApp'));

export function App() {
  return (
    <BrowserRouter>
      <div className="app">
        <Navigation />
        
        <main>
          <ErrorBoundary>
            <Suspense fallback={<div>Loading...</div>}>
              <Routes>
                <Route path="/" element={<HomePage />} />
                <Route path="/products/*" element={<ProductApp />} />
                <Route path="/checkout/*" element={<CheckoutApp />} />
              </Routes>
            </Suspense>
          </ErrorBoundary>
        </main>
      </div>
    </BrowserRouter>
  );
}

// src/components/Navigation.tsx
import { Link } from 'react-router-dom';

export function Navigation() {
  return (
    <nav className="navigation">
      <Link to="/">Home</Link>
      <Link to="/products">Products</Link>
      <Link to="/checkout">Checkout</Link>
    </nav>
  );
}

// src/components/ErrorBoundary.tsx
export class ErrorBoundary extends Component<{children: ReactNode}> {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('MFE Error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
        </div>
      );
    }

    return this.props.children;
  }
}

// webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        productMFE: 'productMFE@https://products.example.com/remoteEntry.js',
        checkoutMFE: 'checkoutMFE@https://checkout.example.com/remoteEntry.js'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        'react-router-dom': { singleton: true }
      }
    })
  ]
};
```

### Example 2: Shared State Between MFEs

```typescript
// ============================================
// Shared Store Package
// ============================================

// packages/shared-store/src/cartStore.ts
import create from 'zustand';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  getTotal: () => number;
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],

  addItem: (item) => {
    set((state) => {
      const existingItem = state.items.find(i => i.id === item.id);
      if (existingItem) {
        return {
          items: state.items.map(i =>
            i.id === item.id
              ? { ...i, quantity: i.quantity + item.quantity }
              : i
          )
        };
      }
      return { items: [...state.items, item] };
    });
  },

  removeItem: (id) => {
    set((state) => ({
      items: state.items.filter(i => i.id !== id)
    }));
  },

  updateQuantity: (id, quantity) => {
    set((state) => ({
      items: state.items.map(i =>
        i.id === id ? { ...i, quantity } : i
      )
    }));
  },

  clearCart: () => set({ items: [] }),

  getTotal: () => {
    return get().items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    );
  }
}));

// Product MFE uses shared store
import { useCartStore } from '@company/shared-store';

function ProductCard({ product }: { product: Product }) {
  const addItem = useCartStore(state => state.addItem);

  const handleAddToCart = () => {
    addItem({
      id: product.id,
      name: product.name,
      price: product.price,
      quantity: 1
    });
  };

  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={handleAddToCart}>Add to Cart</button>
    </div>
  );
}

// Checkout MFE uses same shared store
import { useCartStore } from '@company/shared-store';

function CartSummary() {
  const items = useCartStore(state => state.items);
  const total = useCartStore(state => state.getTotal());

  return (
    <div className="cart-summary">
      <h2>Your Cart</h2>
      {items.map(item => (
        <div key={item.id}>
          {item.name} x {item.quantity} = ${item.price * item.quantity}
        </div>
      ))}
      <div className="total">Total: ${total.toFixed(2)}</div>
    </div>
  );
}

// Shared in Module Federation config
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      shared: {
        '@company/shared-store': { singleton: true, eager: true }
      }
    })
  ]
};
```

### Example 3: Cross-MFE Communication

```typescript
// ============================================
// Event Bus for Communication
// ============================================

// packages/event-bus/src/index.ts
type EventCallback = (data: any) => void;

class EventBus {
  private events: Map<string, EventCallback[]> = new Map();

  subscribe(event: string, callback: EventCallback): () => void {
    if (!this.events.has(event)) {
      this.events.set(event, []);
    }
    this.events.get(event)!.push(callback);

    // Return unsubscribe function
    return () => {
      const callbacks = this.events.get(event);
      if (callbacks) {
        const index = callbacks.indexOf(callback);
        if (index > -1) {
          callbacks.splice(index, 1);
        }
      }
    };
  }

  publish(event: string, data?: any): void {
    const callbacks = this.events.get(event);
    if (callbacks) {
      callbacks.forEach(callback => callback(data));
    }
  }

  clear(): void {
    this.events.clear();
  }
}

export const eventBus = new EventBus();

// Product MFE publishes events
import { eventBus } from '@company/event-bus';

function ProductCard({ product }: { product: Product }) {
  const handleAddToCart = () => {
    eventBus.publish('cart:item-added', {
      productId: product.id,
      name: product.name,
      price: product.price
    });
  };

  return (
    <button onClick={handleAddToCart}>Add to Cart</button>
  );
}

// Navigation MFE subscribes to events
import { eventBus } from '@company/event-bus';

function CartIndicator() {
  const [itemCount, setItemCount] = useState(0);

  useEffect(() => {
    const unsubscribe = eventBus.subscribe('cart:item-added', () => {
      setItemCount(prev => prev + 1);
    });

    return unsubscribe;
  }, []);

  return <div className="cart-indicator">Cart ({itemCount})</div>;
}
```

## Common Mistakes

### 1. Sharing Too Much State

```typescript
// BAD: Tightly coupled through shared state
// Every MFE imports large shared store
import { useGlobalStore } from '@company/shared-store';

// Product MFE
const { user, cart, orders, notifications, settings } = useGlobalStore();

// GOOD: Minimal shared state
// Each MFE has its own state, shares only essentials
import { useAuth } from '@company/shared-auth';  // Only auth
import { useCart } from '@company/shared-cart';  // Only cart

const { user } = useAuth();
const { addToCart } = useCart();
```

### 2. Not Handling MFE Failures

```typescript
// BAD: No error handling
const ProductApp = lazy(() => import('productMFE/ProductApp'));

// App crashes if MFE fails to load

// GOOD: Graceful degradation
const ProductApp = lazy(() =>
  import('productMFE/ProductApp').catch(() => {
    return { default: () => <div>Products unavailable</div> };
  })
);

function App() {
  return (
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <Suspense fallback={<div>Loading...</div>}>
        <ProductApp />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### 3. Duplicating Dependencies

```typescript
// BAD: Each MFE bundles React independently
// productMFE: react (18.0.0) - 130KB
// checkoutMFE: react (18.0.0) - 130KB
// navMFE: react (18.0.0) - 130KB
// Total: 390KB of duplicate React

// GOOD: Share dependencies via Module Federation
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      shared: {
        react: { singleton: true, eager: true },
        'react-dom': { singleton: true, eager: true }
      }
    })
  ]
};
// React loaded once, shared across all MFEs
```

## Best Practices

### 1. Define Clear Contracts

```typescript
// MFE exports clear interface
// product-mfe/src/types.ts
export interface ProductMFEProps {
  apiUrl?: string;
  theme?: 'light' | 'dark';
  onNavigate?: (path: string) => void;
}

export interface ProductMFEExports {
  mount: (container: HTMLElement, props?: ProductMFEProps) => void;
  unmount: (container: HTMLElement) => void;
}

// Shell knows what to expect
import type { ProductMFEExports } from 'productMFE/types';

const ProductMFE: ProductMFEExports = await import('productMFE');
ProductMFE.mount(container, { theme: 'dark' });
```

### 2. Version Compatible Shared Dependencies

```javascript
// Careful with version compatibility
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',  // Specify version
          strictVersion: false  // Allow compatible versions
        }
      }
    })
  ]
};
```

### 3. Implement Health Checks

```typescript
// MFE health check
export async function checkHealth(): Promise<boolean> {
  try {
    await fetch('/api/products/health');
    return true;
  } catch {
    return false;
  }
}

// Shell monitors health
async function loadMFEWithHealthCheck(name: string, url: string) {
  const health = await fetch(`${url}/health`);
  if (!health.ok) {
    console.warn(`MFE ${name} unhealthy, using fallback`);
    return FallbackComponent;
  }
  return await loadMFE(url);
}
```

### 4. Use Feature Flags

```typescript
// Control MFE availability
const FEATURE_FLAGS = {
  productMFE: true,
  checkoutMFE: process.env.ENABLE_CHECKOUT === 'true',
  experimentalMFE: false
};

function App() {
  return (
    <Routes>
      {FEATURE_FLAGS.productMFE && (
        <Route path="/products/*" element={<ProductApp />} />
      )}
      {FEATURE_FLAGS.checkoutMFE && (
        <Route path="/checkout/*" element={<CheckoutApp />} />
      )}
    </Routes>
  );
}
```

## When to Use/Not to Use

### Use Micro Frontends When:

1. **Large Teams**: Multiple teams working on same product
2. **Independent Deployment**: Teams need to deploy independently
3. **Different Technologies**: Want to use different frameworks
4. **Clear Boundaries**: Application has distinct domains
5. **Long-Term Project**: Benefits justify complexity

### Don't Use When:

1. **Small Teams**: Overhead not worth it for small teams
2. **Simple Applications**: Complexity exceeds benefits
3. **Tight Integration**: Features heavily interdependent
4. **Performance Critical**: Can't afford bundle duplication
5. **Limited Resources**: Can't maintain infrastructure

## Interview Questions

### Junior Level

**Q: What are micro frontends?**

A: An architectural pattern where a frontend application is split into multiple smaller, independently deployable applications that work together. Each micro frontend is owned by a different team.

**Q: What is Module Federation?**

A: A Webpack feature that allows loading code dynamically from another separately built and deployed application, enabling true runtime integration of micro frontends.

### Mid Level

**Q: How do you handle shared state in micro frontends?**

A: Options include:
- Shared state management library (Zustand, Redux)
- Event bus for communication
- Props/callbacks through shell
- Browser storage (localStorage, sessionStorage)
- Backend state synchronization

**Q: What are the tradeoffs of micro frontends?**

A: Tradeoffs include:
Benefits: Team autonomy, independent deployment, technology flexibility
Costs: Increased complexity, potential performance overhead, coordination challenges, harder integration testing

### Senior Level

**Q: How do you handle versioning and backward compatibility in micro frontends?**

A: Strategies:
- Semantic versioning for shared dependencies
- Version negotiation in Module Federation
- Backward compatible APIs
- Parallel running of old/new versions
- Feature flags for gradual rollout
- Contract testing between MFEs

**Q: How do you optimize performance in a micro frontend architecture?**

A: Techniques:
- Share dependencies to avoid duplication
- Lazy load MFEs on demand
- Preload critical MFEs
- Use HTTP/2 multiplexing
- CDN deployment
- Code splitting within MFEs
- Monitor bundle sizes
- Implement loading strategies (eager vs lazy)

## Key Takeaways

1. Micro frontends extend microservices concept to frontend
2. Each MFE is independently deployable
3. Module Federation enables runtime integration with shared dependencies
4. Choose integration approach based on needs (build-time, runtime, iframes)
5. Shared state management requires careful design
6. Error boundaries essential for fault isolation
7. Performance considerations due to potential duplication
8. Best for large teams with clear domain boundaries
9. Adds complexity - use when benefits justify costs
10. Requires strong DevOps and infrastructure support

## Resources

### Tools
- Webpack Module Federation: https://webpack.js.org/concepts/module-federation/
- Single-SPA: https://single-spa.js.org/
- Bit: https://bit.dev/
- Nx: https://nx.dev/

### Books
- "Micro Frontends in Action" by Michael Geers
- "Building Micro-Frontends" by Luca Mezzalira

### Articles
- Micro Frontends by Martin Fowler: https://martinfowler.com/articles/micro-frontends.html
- Module Federation Examples: https://github.com/module-federation/module-federation-examples

### Examples
- Module Federation Examples: https://github.com/module-federation/module-federation-examples
- Micro Frontend Frameworks: https://micro-frontends.org/
