# CSR - Client-Side Rendering

## The Idea

**In plain English:** Client-Side Rendering (CSR) is when your browser does all the work of building a webpage after it arrives. The server sends you an almost-empty page and a bag of instructions (JavaScript), and then your browser follows those instructions to actually draw everything you see on screen.

**Real-world analogy:** Imagine ordering a flat-pack furniture kit from a store. The store ships you a box of parts and an instruction booklet, and you assemble the furniture yourself at home.

- The store (server) = the web server that sends the files
- The flat-pack box of parts and instructions = the minimal HTML and JavaScript bundle sent to the browser
- You assembling the furniture at home = the browser executing JavaScript to build and display the page

---

## Overview

Client-Side Rendering (CSR) is a rendering strategy where the browser downloads a minimal HTML document, fetches JavaScript, and then renders the entire UI in the browser. This approach became dominant with the rise of Single Page Applications (SPAs) using frameworks like React, Angular, and Vue.

## How CSR Works

```
Traditional CSR Flow:

1. Browser Request
   └─→ Server returns minimal HTML
        <!DOCTYPE html>
        <html>
          <head>
            <script src="bundle.js"></script>
          </head>
          <body>
            <div id="root"></div>  ← Empty!
          </body>
        </html>

2. Download JavaScript
   └─→ Browser fetches bundle.js (React, app code, etc.)

3. Execute JavaScript
   └─→ React initializes
   └─→ Makes API calls for data
   └─→ Renders components

4. Display Content
   └─→ User sees content

Timeline:
├─ HTML arrives (100ms)
├─ JS downloads (500ms)
├─ JS executes (200ms)
├─ API calls (300ms)
└─ Content visible (1100ms) ← SLOW!
```

## Basic CSR Implementation

### React CSR Example

```typescript
// index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>CSR App</title>
  </head>
  <body>
    <div id="root"></div>
    <!-- React app renders here -->
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>

// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './index.css';

// Mount React app to empty div
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
);

// src/App.tsx
import { Routes, Route } from 'react-router-dom';
import HomePage from './pages/HomePage';
import ProductPage from './pages/ProductPage';
import NotFound from './pages/NotFound';

function App() {
  return (
    <div className="app">
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/products/:id" element={<ProductPage />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </div>
  );
}

export default App;

// src/pages/HomePage.tsx
import { useState, useEffect } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
}

function HomePage() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);

  // Fetch data client-side
  useEffect(() => {
    const fetchProducts = async () => {
      try {
        const response = await fetch('/api/products');
        const data = await response.json();
        setProducts(data);
      } catch (error) {
        console.error('Failed to fetch products:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchProducts();
  }, []);

  if (loading) {
    return <div className="loading">Loading products...</div>;
  }

  return (
    <div className="home">
      <h1>Products</h1>
      <div className="products-grid">
        {products.map((product) => (
          <div key={product.id} className="product-card">
            <h2>{product.name}</h2>
            <p>${product.price}</p>
          </div>
        ))}
      </div>
    </div>
  );
}

export default HomePage;
```

### Angular CSR Example

```typescript
// main.ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';

// Bootstrap Angular app on client
platformBrowserDynamic()
  .bootstrapModule(AppModule)
  .catch((err) => console.error(err));

// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { RouterModule, Routes } from '@angular/router';
import { AppComponent } from './app.component';
import { HomeComponent } from './home/home.component';
import { ProductComponent } from './product/product.component';

const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'products/:id', component: ProductComponent },
  { path: '**', redirectTo: '' }
];

@NgModule({
  declarations: [AppComponent, HomeComponent, ProductComponent],
  imports: [
    BrowserModule,
    HttpClientModule,
    RouterModule.forRoot(routes)
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}

// home.component.ts
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-home',
  template: `
    <div class="home">
      <h1>Products</h1>
      
      <div *ngIf="loading" class="loading">
        Loading products...
      </div>

      <div *ngIf="!loading" class="products-grid">
        <div *ngFor="let product of products" class="product-card">
          <h2>{{ product.name }}</h2>
          <p>\${{ product.price }}</p>
        </div>
      </div>
    </div>
  `
})
export class HomeComponent implements OnInit {
  products: Product[] = [];
  loading = true;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    // Fetch data client-side
    this.http.get<Product[]>('/api/products').subscribe({
      next: (data) => {
        this.products = data;
        this.loading = false;
      },
      error: (err) => {
        console.error('Failed to fetch products:', err);
        this.loading = false;
      }
    });
  }
}
```

## Single Page Application (SPA) Pattern

### Client-Side Routing

```typescript
// React Router for client-side navigation
import { 
  BrowserRouter, 
  Routes, 
  Route, 
  Link, 
  useNavigate 
} from 'react-router-dom';

function Navigation() {
  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/about">About</Link>
      <Link to="/products">Products</Link>
    </nav>
  );
}

function App() {
  return (
    <BrowserRouter>
      <Navigation />
      <Routes>
        {/* No page reload on navigation */}
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/products" element={<Products />} />
        <Route path="/products/:id" element={<ProductDetail />} />
      </Routes>
    </BrowserRouter>
  );
}

// Programmatic navigation
function ProductCard({ id }: { id: number }) {
  const navigate = useNavigate();

  const handleClick = () => {
    // Navigate without page reload
    navigate(`/products/${id}`);
  };

  return (
    <div onClick={handleClick} className="product-card">
      View Product {id}
    </div>
  );
}
```

### State Management in CSR

```typescript
// React Context for global state
import { createContext, useContext, useState, ReactNode } from 'react';

interface User {
  id: number;
  name: string;
  email: string;
}

interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  loading: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);

  const login = async (email: string, password: string) => {
    setLoading(true);
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
      });

      const data = await response.json();
      setUser(data.user);
      localStorage.setItem('token', data.token);
    } catch (error) {
      console.error('Login failed:', error);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const logout = () => {
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Usage in component
function LoginPage() {
  const { login, loading } = useAuth();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await login(email, password);
      // Redirect after login (client-side)
      window.location.href = '/dashboard';
    } catch (error) {
      alert('Login failed');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

## Data Fetching Patterns

### Loading States

```typescript
// React component with proper loading states
import { useState, useEffect } from 'react';

type DataState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function ProductList() {
  const [state, setState] = useState<DataState<Product[]>>({ 
    status: 'idle' 
  });

  useEffect(() => {
    setState({ status: 'loading' });

    fetch('/api/products')
      .then((res) => res.json())
      .then((data) => setState({ status: 'success', data }))
      .catch((error) => setState({ status: 'error', error }));
  }, []);

  // Render based on state
  switch (state.status) {
    case 'idle':
      return null;
    
    case 'loading':
      return (
        <div className="loading">
          <Spinner />
          <p>Loading products...</p>
        </div>
      );
    
    case 'error':
      return (
        <div className="error">
          <p>Failed to load products: {state.error.message}</p>
          <button onClick={() => window.location.reload()}>
            Retry
          </button>
        </div>
      );
    
    case 'success':
      return (
        <div className="products">
          {state.data.map((product) => (
            <ProductCard key={product.id} product={product} />
          ))}
        </div>
      );
  }
}
```

### React Query for Data Fetching

```typescript
// Modern data fetching with React Query
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Fetch products
function ProductList() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['products'],
    queryFn: async () => {
      const response = await fetch('/api/products');
      if (!response.ok) throw new Error('Failed to fetch products');
      return response.json();
    },
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
    retry: 3
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <button onClick={() => refetch()}>Refresh</button>
      <div className="products">
        {data.map((product: Product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}

// Mutation for creating product
function CreateProduct() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: async (newProduct: Omit<Product, 'id'>) => {
      const response = await fetch('/api/products', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newProduct)
      });
      return response.json();
    },
    onSuccess: () => {
      // Invalidate and refetch products
      queryClient.invalidateQueries({ queryKey: ['products'] });
    }
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    mutation.mutate({ name: 'New Product', price: 99.99 });
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Product'}
      </button>
      {mutation.isError && <div>Error: {mutation.error.message}</div>}
    </form>
  );
}
```

## Optimizing CSR Performance

### Code Splitting

```typescript
// React lazy loading for code splitting
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Lazy load route components
const Home = lazy(() => import('./pages/Home'));
const Products = lazy(() => import('./pages/Products'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<div>Loading page...</div>}>
      <Routes>
        {/* Each route is a separate chunk */}
        <Route path="/" element={<Home />} />
        <Route path="/products" element={<Products />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}

// Component-level code splitting
function ProductPage() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <h1>Product Analytics</h1>
      <button onClick={() => setShowChart(true)}>
        Show Chart
      </button>

      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          {/* Heavy chart library loaded only when needed */}
          <LazyChart />
        </Suspense>
      )}
    </div>
  );
}

const LazyChart = lazy(() => import('./components/Chart'));
```

### Prefetching Data

```typescript
// Prefetch data on hover
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { Link } from 'react-router-dom';

function ProductLink({ productId }: { productId: number }) {
  const queryClient = useQueryClient();

  const prefetchProduct = () => {
    queryClient.prefetchQuery({
      queryKey: ['product', productId],
      queryFn: async () => {
        const response = await fetch(`/api/products/${productId}`);
        return response.json();
      }
    });
  };

  return (
    <Link
      to={`/products/${productId}`}
      onMouseEnter={prefetchProduct} // Prefetch on hover
      onTouchStart={prefetchProduct} // Prefetch on touch
    >
      View Product {productId}
    </Link>
  );
}

// Product detail page has data ready
function ProductDetail({ productId }: { productId: number }) {
  const { data, isLoading } = useQuery({
    queryKey: ['product', productId],
    queryFn: async () => {
      const response = await fetch(`/api/products/${productId}`);
      return response.json();
    }
  });

  // Data might already be cached from prefetch
  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>${data.price}</p>
    </div>
  );
}
```

### Virtual Scrolling

```typescript
// Efficient rendering of large lists
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function VirtualProductList({ products }: { products: Product[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: products.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100, // Estimated item height
    overscan: 5 // Render 5 extra items above/below viewport
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '600px', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative'
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const product = products[virtualItem.index];
          
          return (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`
              }}
            >
              <ProductCard product={product} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## CSR Architecture Patterns

```
┌────────────────────────────────────────────────────────────┐
│                   CSR Application Architecture              │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Browser                                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Initial Load                                         │  │
│  │  ├─ Download minimal HTML                             │  │
│  │  ├─ Download JavaScript bundles                       │  │
│  │  ├─ Execute React/Angular/Vue                         │  │
│  │  └─ Render empty shell                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Data Fetching                                        │  │
│  │  ├─ API calls to backend                              │  │
│  │  ├─ Wait for responses                                │  │
│  │  └─ Update UI with data                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  User Interaction                                     │  │
│  │  ├─ Client-side routing (no page reload)             │  │
│  │  ├─ State updates                                     │  │
│  │  ├─ API calls                                         │  │
│  │  └─ Re-render components                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

## Pros and Cons of CSR

### Advantages

```typescript
// 1. Rich Interactions
function InteractiveApp() {
  const [count, setCount] = useState(0);
  
  // Instant UI updates, no server roundtrip
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Clicked {count} times
      </button>
      <AnimatedChart data={data} /> {/* Smooth animations */}
      <DragAndDrop /> {/* Complex interactions */}
    </div>
  );
}

// 2. Reduced Server Load
// Server only serves static files and API endpoints
// No HTML rendering on server

// 3. Better for SPAs
// Fast navigation between pages
// Preserve state during navigation
function Navigation() {
  return (
    <Router>
      <Route path="/products" element={<Products />} />
      <Route path="/cart" element={<Cart />} />
      {/* Cart state preserved when navigating to products */}
    </Router>
  );
}

// 4. Easier Development
// Separate frontend and backend
// Can deploy frontend to CDN
// API-first architecture
```

### Disadvantages

```typescript
// 1. Slow Initial Load
// Must download and execute JS before showing content
// Poor Time to Interactive (TTI)

// 2. SEO Challenges
// Search engines see empty HTML
// Need special handling for crawlers

// 3. Performance on Low-End Devices
// Heavy JavaScript execution
// Large bundle sizes

// 4. No Content Without JavaScript
// Users with JS disabled see nothing
// JS errors break the entire app

// 5. Larger Bundle Sizes
// Entire app code shipped to client
import React from 'react'; // ~40KB
import ReactDOM from 'react-dom'; // ~130KB
import { RouterHistory, Routes } from 'react-router-dom'; // ~50KB
// + Your app code
// + Dependencies
// = 500KB+ JavaScript
```

## Common Mistakes

### 1. No Loading States

```typescript
// ❌ BAD: No loading indicator
function ProductList() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetch('/api/products')
      .then((res) => res.json())
      .then(setProducts);
  }, []);

  return (
    <div>
      {/* Shows nothing while loading */}
      {products.map((p) => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}

// ✅ GOOD: Proper loading state
function ProductList() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/products')
      .then((res) => res.json())
      .then((data) => {
        setProducts(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <LoadingSpinner />;

  return (
    <div>
      {products.map((p) => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

### 2. Not Handling Errors

```typescript
// ❌ BAD: No error handling
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then(setUser);
    // What if this fails?
  }, [userId]);

  return <div>{user?.name}</div>;
}

// ✅ GOOD: Proper error handling
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    setError(null);

    fetch(`/api/users/${userId}`)
      .then((res) => {
        if (!res.ok) throw new Error('Failed to fetch user');
        return res.json();
      })
      .then((data) => {
        setUser(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return <div>User not found</div>;

  return <div>{user.name}</div>;
}
```

### 3. Memory Leaks

```typescript
// ❌ BAD: Memory leak
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then(setUser); // Calls setState after unmount!
  }, [userId]);

  return <div>{user?.name}</div>;
}

// ✅ GOOD: Cleanup with AbortController
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => res.json())
      .then(setUser)
      .catch((err) => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });

    return () => controller.abort(); // Cleanup
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

## Best Practices

### 1. Implement Skeleton Screens

```typescript
// Show content structure while loading
function ProductListSkeleton() {
  return (
    <div className="products-grid">
      {[...Array(6)].map((_, i) => (
        <div key={i} className="product-card skeleton">
          <div className="skeleton-image" />
          <div className="skeleton-title" />
          <div className="skeleton-price" />
        </div>
      ))}
    </div>
  );
}

function ProductList() {
  const { data, isLoading } = useQuery(['products'], fetchProducts);

  if (isLoading) return <ProductListSkeleton />;

  return (
    <div className="products-grid">
      {data.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

### 2. Optimize Bundle Size

```typescript
// Use dynamic imports
const HeavyComponent = lazy(() => import('./HeavyComponent'));

// Tree-shake unused code
import { Button } from 'components'; // ✅ Named import
// vs
import * as Components from 'components'; // ❌ Imports everything

// Use smaller alternatives
import dayjs from 'dayjs'; // ✅ 2KB
// vs
import moment from 'moment'; // ❌ 70KB
```

### 3. Implement Progressive Enhancement

```typescript
// Show basic content, enhance with JS
function ProductCard({ product }: { product: Product }) {
  return (
    <div className="product-card">
      {/* Always visible */}
      <h2>{product.name}</h2>
      <p>${product.price}</p>
      
      {/* Enhanced with JS */}
      <AddToCartButton productId={product.id} />
      <ProductRating productId={product.id} />
    </div>
  );
}
```

## When to Use CSR

### Use CSR When

1. **Building SPAs**: Complex, interactive web applications
2. **Private Dashboards**: Authentication-required admin panels
3. **Real-time Apps**: Chat, collaboration tools
4. **Rich Interactions**: Heavy client-side logic, animations
5. **API-First Architecture**: Decoupled frontend/backend

### Don't Use CSR When

1. **SEO is Critical**: Marketing sites, blogs, e-commerce
2. **Performance is Priority**: Need fast initial load
3. **Content-Heavy Sites**: News, documentation
4. **Low-End Devices**: Target users with slow devices
5. **Accessibility is Critical**: Need content without JS

## Interview Questions

### 1. What is Client-Side Rendering and how does it work?
**Answer**: CSR is when the browser downloads a minimal HTML document with a JavaScript bundle, then executes JavaScript to render the UI. The initial HTML is nearly empty with a root div. Once JS loads and executes, it makes API calls to fetch data and renders components. This differs from server rendering where HTML is generated on the server.

### 2. What are the main disadvantages of CSR?
**Answer**: Slow initial load (must download and execute JS before showing content), poor SEO (search engines see empty HTML), performance issues on low-end devices (heavy JS execution), no content without JavaScript, and larger bundle sizes. Time to Interactive (TTI) is typically slower than SSR.

### 3. How do you optimize CSR performance?
**Answer**: Code splitting with dynamic imports, lazy loading routes and components, prefetching data on hover, virtual scrolling for large lists, bundle optimization (tree shaking), caching with React Query, skeleton screens for perceived performance, and CDN deployment for static assets.

### 4. What is the difference between CSR and SSR?
**Answer**: CSR renders on the client after JS loads, SSR renders HTML on the server. CSR has slower initial load but faster subsequent navigation. SSR has better SEO and faster Time to First Byte (TTFB). CSR requires JS to work, SSR provides content without JS. CSR reduces server load, SSR increases server load but improves perceived performance.

### 5. How do you handle SEO in CSR applications?
**Answer**: Use prerendering for static pages, implement dynamic rendering (serve pre-rendered HTML to bots), use hybrid approaches like Next.js with ISR, add proper meta tags and structured data, implement server-side rendering for critical pages, or use services like Prerender.io. For SPAs, consider using frameworks that support SSR/SSG.

## Key Takeaways

1. **CSR Renders in Browser**: All rendering happens client-side after JavaScript loads.
2. **Great for SPAs**: Ideal for highly interactive, authenticated applications.
3. **Slow Initial Load**: Trade-off between initial load and subsequent performance.
4. **SEO Challenges**: Difficult for search engines to crawl and index.
5. **Code Splitting is Essential**: Break large bundles into smaller chunks.
6. **Loading States Required**: Always show loading indicators for better UX.
7. **Not for Everyone**: Consider SSR/SSG for content-heavy, SEO-critical sites.
8. **Optimize Aggressively**: Bundle size and load time are critical in CSR.

## Resources

- [React Documentation](https://react.dev/)
- [Angular Documentation](https://angular.io/)
- [Vue.js Documentation](https://vuejs.org/)
- [React Router](https://reactrouter.com/)
- [TanStack Query (React Query)](https://tanstack.com/query)
- [Web.dev - Rendering on the Web](https://web.dev/rendering-on-the-web/)
- [MDN - Client-side JavaScript Frameworks](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks)
