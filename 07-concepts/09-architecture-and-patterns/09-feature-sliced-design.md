# Feature Sliced Design (FSD)

## The Idea

**In plain English:** Feature Sliced Design is a way of organizing all the files in a web app so that each piece of code lives in a clearly labelled "floor" of a building, and code on a higher floor can call down to lower floors, but never the other way around. This keeps features from becoming tangled messes where changing one thing accidentally breaks another.

**Real-world analogy:** Think of a large department store with six floors. The ground floor (utilities) sells basic supplies anyone can use. As you go up, each floor sells more specialized things built from what's below — electronics uses cables from the ground floor, but the cables section never needs to stock TVs. Each department on the same floor runs its own checkout and never borrows stock from the department next door.

- The ground floor (Supplies) = `shared/` — generic utilities every part of the app can use
- The middle floors (Departments) = `entities/`, `features/`, `widgets/` — business objects, user actions, and composed UI sections
- The top floor (Store Management) = `pages/` and `app/` — the overall layout that assembles everything below
- Each floor only restocking from floors below = the one-directional import rule (no upward imports)
- Each department running its own checkout = slice isolation (same-floor sections cannot import from each other)

---

## Overview

Feature Sliced Design (FSD) is a frontend architectural methodology that organizes code by business features and layers, with strict rules about which layers can depend on which. Unlike component-library-focused methodologies (Atomic Design), FSD addresses the full application architecture — where to put API calls, state, business logic, pages, and shared utilities. It's particularly effective for medium-to-large applications where feature isolation and maintainability are critical.

## FSD Layers

```
FSD Layer Hierarchy (top to bottom, high to low level):

app/          Application initialization, providers, global styles, routing
  └─ Can import from: pages, widgets, features, entities, shared

pages/        Route-level pages; assembles widgets/features for a route
  └─ Can import from: widgets, features, entities, shared

widgets/      Self-contained UI blocks (independent from pages)
  └─ Can import from: features, entities, shared

features/     User interactions, actions that change application state
  └─ Can import from: entities, shared

entities/     Business domain objects (User, Post, Cart, Order)
  └─ Can import from: shared

shared/       Reusable utilities, UI kit, API clients, constants
  └─ Can import from: nothing above it (pure utilities)

RULE: Each layer can only import from layers BELOW it in the hierarchy.
      No upward imports allowed. No same-level imports (within slices).
```

## Layer Definitions

### shared/

No business logic — pure utilities, design system, API configuration:

```
shared/
├── ui/          Design system components (Button, Input, Modal, Badge)
├── api/         HTTP client configuration, base API wrapper
├── lib/         Generic utilities (date formatting, validation helpers)
├── config/      Environment config, constants
├── types/       Shared TypeScript types
└── hooks/       Generic React hooks (useDebounce, useIntersectionObserver)
```

```typescript
// shared/api/base.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: process.env.VITE_API_URL,
  timeout: 10_000,
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// shared/lib/format-date.ts
export function formatDate(date: Date, locale = 'en-US'): string {
  return new Intl.DateTimeFormat(locale).format(date);
}

// shared/ui/Button/Button.tsx
// Generic, no business logic
export function Button({ children, ...props }: ButtonProps) {
  return <button {...props}>{children}</button>;
}
```

### entities/

Business domain objects — their data model, UI representation, and API access:

```
entities/
├── user/
│   ├── api/         userApi.ts — CRUD for User
│   ├── model/       userStore.ts, userTypes.ts, userSelectors.ts
│   ├── ui/          UserAvatar.tsx, UserCard.tsx
│   └── index.ts     Public API (re-exports only what's intended public)
│
├── product/
│   ├── api/         productApi.ts
│   ├── model/       productStore.ts, productTypes.ts
│   ├── ui/          ProductCard.tsx, ProductPrice.tsx
│   └── index.ts
│
└── cart/
    ├── api/         cartApi.ts
    ├── model/       cartStore.ts, cartSelectors.ts
    └── index.ts
```

```typescript
// entities/product/api/productApi.ts
import { apiClient } from '@/shared/api/base';
import { Product } from '../model/productTypes';

export const productApi = {
  getById: (id: string): Promise<Product> =>
    apiClient.get(`/products/${id}`).then((r) => r.data),

  list: (params: ListProductsParams): Promise<Product[]> =>
    apiClient.get('/products', { params }).then((r) => r.data),
};

// entities/product/model/productTypes.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  categoryId: string;
  imageUrl: string;
  inStock: boolean;
}

// entities/product/ui/ProductCard.tsx
// ProductCard displays a product — no domain logic, just rendering
import { Product } from '../model/productTypes';
import { formatCurrency } from '@/shared/lib/format-currency';

export function ProductCard({ product }: { product: Product }) {
  return (
    <article>
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <span>{formatCurrency(product.price)}</span>
    </article>
  );
}

// entities/product/index.ts — Public API
// ONLY export what other layers should use
export { ProductCard } from './ui/ProductCard';
export { productApi } from './api/productApi';
export type { Product } from './model/productTypes';
// Internal model details, helper functions NOT exported
```

### features/

User interactions and actions that change application state. Features are verbs — they represent user capabilities:

```
features/
├── add-to-cart/       (verb: "add to cart")
│   ├── api/
│   ├── model/
│   ├── ui/            AddToCartButton.tsx
│   └── index.ts
│
├── auth/
│   ├── sign-in/       SignInForm.tsx, signInMutation.ts
│   ├── sign-up/
│   └── sign-out/
│
├── product-filter/    FilterPanel.tsx, filterModel.ts
├── product-search/    SearchInput.tsx, searchModel.ts
└── checkout/          CheckoutForm.tsx, checkoutMutation.ts
```

```typescript
// features/add-to-cart/ui/AddToCartButton.tsx
// Imports from entities (below) and shared (below) — never from features/ siblings
import { Product } from '@/entities/product';
import { cartApi } from '@/entities/cart';
import { Button, Spinner } from '@/shared/ui';

export function AddToCartButton({ product }: { product: Product }) {
  const { mutate, isPending } = useMutation({
    mutationFn: (quantity: number) =>
      cartApi.addItem({ productId: product.id, quantity }),
    onSuccess: () => toast.success(`${product.name} added to cart`),
  });

  return (
    <Button
      onClick={() => mutate(1)}
      disabled={!product.inStock || isPending}
    >
      {isPending ? <Spinner /> : 'Add to Cart'}
    </Button>
  );
}

// features/add-to-cart/index.ts
export { AddToCartButton } from './ui/AddToCartButton';
```

### widgets/

Self-contained UI blocks that combine features and entities for a specific section of the UI:

```
widgets/
├── header/            Header.tsx — Logo + Nav + SearchBar + UserMenu
├── product-catalog/   ProductCatalog.tsx — ProductGrid + FilterPanel
├── cart-sidebar/      CartSidebar.tsx — CartItems + CartSummary
└── footer/            Footer.tsx
```

```typescript
// widgets/product-catalog/ui/ProductCatalog.tsx
// Combines feature (filter) with entity display (product cards)
import { ProductCard, productApi } from '@/entities/product';
import { ProductFilter, useProductFilter } from '@/features/product-filter';
import { AddToCartButton } from '@/features/add-to-cart';

export function ProductCatalog() {
  const { filter, setFilter } = useProductFilter();
  const { data: products } = useQuery({
    queryKey: ['products', filter],
    queryFn: () => productApi.list(filter),
  });

  return (
    <section>
      <ProductFilter value={filter} onChange={setFilter} />
      <div className="grid">
        {products?.map((product) => (
          <div key={product.id}>
            <ProductCard product={product} />
            <AddToCartButton product={product} />
          </div>
        ))}
      </div>
    </section>
  );
}
```

### pages/

Route-level composition — assembles widgets for a specific route:

```
pages/
├── home/
│   ├── ui/          HomePage.tsx
│   └── index.ts
├── product-detail/
│   ├── ui/          ProductDetailPage.tsx
│   └── index.ts
└── checkout/
    ├── ui/          CheckoutPage.tsx
    └── index.ts
```

```typescript
// pages/product-detail/ui/ProductDetailPage.tsx
import { Header } from '@/widgets/header';
import { ProductCatalog } from '@/widgets/product-catalog';
import { CartSidebar } from '@/widgets/cart-sidebar';
import { Footer } from '@/widgets/footer';

export function ProductDetailPage() {
  const { productId } = useParams<{ productId: string }>();

  return (
    <>
      <Header />
      <main>
        <ProductDetailWidget productId={productId!} />
        <CartSidebar />
      </main>
      <Footer />
    </>
  );
}
```

### app/

Application bootstrap — providers, routing, global error boundaries:

```typescript
// app/providers/AppProviders.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider } from '@/shared/ui/theme';

const queryClient = new QueryClient();

export function AppProviders({ children }: { children: ReactNode }) {
  return (
    <BrowserRouter>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </QueryClientProvider>
    </BrowserRouter>
  );
}

// app/router/AppRouter.tsx
import { Routes, Route } from 'react-router-dom';
import { HomePage } from '@/pages/home';
import { ProductDetailPage } from '@/pages/product-detail';
import { CheckoutPage } from '@/pages/checkout';

export function AppRouter() {
  return (
    <Routes>
      <Route path="/" element={<HomePage />} />
      <Route path="/products/:productId" element={<ProductDetailPage />} />
      <Route path="/checkout" element={<CheckoutPage />} />
    </Routes>
  );
}
```

## Slices: Isolation Within Layers

Within features, entities, and pages, each folder is a "slice" — they cannot import from each other at the same level:

```
RULE: Same-level slices are isolated from each other.

features/add-to-cart/  → cannot import from features/product-filter/
features/product-filter/ → cannot import from features/add-to-cart/

entities/product/ → cannot import from entities/cart/
entities/cart/    → cannot import from entities/product/
```

This isolation prevents circular dependencies and makes slices independently deployable and testable.

```typescript
// ❌ Violation: same-level slice import
// features/add-to-cart/ui/AddToCartButton.tsx
import { useProductFilter } from '@/features/product-filter'; // ← VIOLATION

// ✅ Communicate via shared state (store) or through a higher layer
// If AddToCart needs to know about filters, that logic belongs in a widget
// or page that composes both features
```

## Public API via index.ts

Each slice exposes a public API through its `index.ts`. Internal implementation details stay private:

```typescript
// entities/user/index.ts — public API

// ✅ Export what other layers need
export { UserAvatar } from './ui/UserAvatar';
export { userApi } from './api/userApi';
export { useCurrentUser } from './model/userHooks';
export type { User, UserRole } from './model/userTypes';

// ✅ Do NOT export internal helpers:
// export { getUserDisplayName } from './lib/getUserDisplayName'; ← internal
// export { userStore } from './model/userStore'; ← internal detail
```

## FSD vs Feature-Based Architecture vs Atomic Design

```
Architecture comparison:

Feature-based (colocation):
  features/
    product-list/
      ProductList.tsx      (mixes entity, feature, widget concerns)
      productListApi.ts
      productListStore.ts
  Problem: no clear rules about what goes where;
           cross-feature dependencies grow unchecked

Atomic Design (component hierarchy):
  atoms/ molecules/ organisms/
  Addresses: component design
  Doesn't address: where API logic lives, state management, business logic

Feature Sliced Design:
  Explicit dependency direction (layers)
  Explicit isolation rules (slices)
  Addresses: full app architecture including state, API, business logic
  Overhead: more directories, more discipline required
```

## When to Use FSD

```
Use FSD when:
  ✓ Large application (50+ features, multiple teams)
  ✓ Long-term maintenance is critical
  ✓ Team size > 5 developers
  ✓ Features frequently change independently
  ✓ Multiple products sharing entities (monorepo)

Prefer simpler architecture when:
  ✗ Small application (< 20 features)
  ✗ Solo developer or very small team
  ✗ Short-lived application or MVP
  ✗ Team unfamiliar with FSD (high learning curve)
```

## Common Mistakes

### 1. Importing Upward in the Hierarchy

```typescript
// ❌ shared importing from entities — upward dependency!
// shared/ui/CartBadge.tsx
import { useCartCount } from '@/entities/cart'; // ← VIOLATION

// ✅ CartBadge receives count as a prop
// The entity-aware version lives in entities/cart/ui/CartBadge.tsx
// The generic badge lives in shared/ui/Badge.tsx
```

### 2. Putting Business Logic in shared/

```typescript
// ❌ Business logic in shared/
// shared/lib/checkout-calculations.ts
export function calculateFinalPrice(cart: Cart, coupon: Coupon): number {
  // Business logic here — depends on Cart and Coupon types
}

// ✅ Business logic belongs in entities or features
// entities/cart/model/cartCalculations.ts
// or features/checkout/model/priceCalculations.ts
```

### 3. Not Using Public APIs (index.ts)

```typescript
// ❌ Deep import — bypasses public API
import { UserCard } from '@/entities/user/ui/UserCard/UserCard';

// ✅ Import from public API
import { UserCard } from '@/entities/user';
// This way, internal restructuring doesn't break consumers
```

## Interview Questions

### 1. What is the core rule of Feature Sliced Design?

**Answer:** The core rule is unidirectional dependencies: each layer can only import from layers below it in the hierarchy (app → pages → widgets → features → entities → shared). No upward imports. Within the same layer, slices (individual feature folders, entity folders) cannot import from each other. This prevents circular dependencies, makes features independently modifiable without ripple effects, and creates a clear mental model of what depends on what.

### 2. What is the difference between an entity and a feature in FSD?

**Answer:** An entity represents a business domain object — the noun. `Product`, `User`, `Cart`, `Order` are entities. They contain the data model, API calls for CRUD operations, and presentational components for displaying that object. A feature represents a user action — the verb. "Add to cart," "filter products," "sign in," "submit review" are features. Features coordinate entities to implement a user capability; they have a mutation, a form, or an interactive component. Entities are passive (display data); features are active (change data in response to user action).

### 3. How does FSD handle shared state between features?

**Answer:** Features at the same level (same layer) cannot import from each other — this is the slice isolation rule. Shared state is handled in one of three ways: (1) lift state to a higher layer — a widget or page that uses both features passes state via props; (2) use a shared store — state lives in an entity or a shared store that both features read from (the store is the communication channel, not a direct import); (3) use React context or a global state manager at the app level for truly global state. The key is that the communication goes through a lower layer (shared/entities) or a higher layer (widget/page), never directly between same-level slices.

### 4. When would you choose FSD over a simpler feature-based structure?

**Answer:** FSD when the application is large (50+ features), the team is large (5+ developers), and you need explicit rules to prevent dependency chaos as the codebase grows. The discipline of explicit layer rules, slice isolation, and public APIs prevents the "big ball of mud" that feature-based structures tend toward over time as developers take shortcuts. For small applications, a simpler flat or feature-colocation approach is more practical — FSD's ceremony (multiple index.ts files, strict layer rules, slice isolation) outweighs its benefit when the application is simple enough that dependencies are obvious without formal rules.
