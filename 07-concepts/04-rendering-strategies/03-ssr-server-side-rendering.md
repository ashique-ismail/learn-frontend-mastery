# SSR - Server-Side Rendering

## The Idea

**In plain English:** Server-Side Rendering (SSR) is when a website's pages are built on the server computer before they reach your browser, so your browser receives a fully finished page instead of having to build it yourself. Think of it as getting a cooked meal delivered versus receiving raw ingredients you have to cook at home.

**Real-world analogy:** Imagine ordering a custom sandwich from a deli. The deli (server) prepares your exact sandwich based on your order, then hands it to you fully made. You can eat it right away (see the content), but you still need to grab your own cutlery and condiments before you can fully enjoy it (the JavaScript "hydration" step that makes the page interactive).

- The deli = the server (builds the HTML before sending it)
- Your sandwich order = the browser's page request
- The fully made sandwich you receive = the complete HTML page sent to your browser
- Grabbing cutlery and condiments after = hydration (JavaScript attaching event listeners to make the page interactive)

---

## Overview

Server-Side Rendering (SSR) is a rendering strategy where HTML is generated on the server for each request, sent to the browser fully formed, and then "hydrated" with JavaScript to become interactive. This provides fast initial page loads and better SEO compared to Client-Side Rendering.

## How SSR Works

```
SSR Flow:

1. Browser Request
   User requests /products
        ↓
2. Server Processing
   ├─ Execute React/Angular components on server
   ├─ Fetch required data from database/API
   ├─ Render components to HTML string
   └─ Inject data into HTML
        ↓
3. Send Full HTML
   Server sends complete HTML document
   <!DOCTYPE html>
   <html>
     <body>
       <div id="root">
         <h1>Products</h1>
         <div class="product">Product 1</div>
         <div class="product">Product 2</div>
       </div>
       <script src="bundle.js"></script>
     </body>
   </html>
        ↓
4. Browser Display
   User sees content immediately
        ↓
5. Hydration
   JavaScript downloads and attaches event listeners
   App becomes interactive

Timeline:
├─ Request sent (0ms)
├─ HTML arrives with content (200ms) ← Fast!
├─ User sees content (200ms) ← TTFB
├─ JS downloads (400ms)
└─ App interactive (600ms) ← TTI
```

## Next.js SSR Implementation

### Basic SSR Page

```typescript
// pages/products/index.tsx
import { GetServerSideProps } from 'next';

interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
}

interface Props {
  products: Product[];
  timestamp: string;
}

// This runs on the server for every request
export const getServerSideProps: GetServerSideProps<Props> = async (context) => {
  // Access request data
  const { req, res, query, params } = context;

  // Set cache headers
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  );

  try {
    // Fetch data on the server
    const response = await fetch('https://api.example.com/products');
    const products = await response.json();

    // Props returned here are passed to the component
    return {
      props: {
        products,
        timestamp: new Date().toISOString()
      }
    };
  } catch (error) {
    // Handle errors
    return {
      props: {
        products: [],
        timestamp: new Date().toISOString()
      }
    };
  }
};

// Component receives props from getServerSideProps
export default function ProductsPage({ products, timestamp }: Props) {
  return (
    <div className="products-page">
      <h1>Products</h1>
      <p>Rendered at: {new Date(timestamp).toLocaleString()}</p>
      
      <div className="products-grid">
        {products.map((product) => (
          <div key={product.id} className="product-card">
            <h2>{product.name}</h2>
            <p>${product.price}</p>
            <p>{product.description}</p>
            <button>Add to Cart</button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Dynamic SSR Pages

```typescript
// pages/products/[id].tsx
import { GetServerSideProps } from 'next';

interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
  images: string[];
}

interface Props {
  product: Product | null;
  relatedProducts: Product[];
}

export const getServerSideProps: GetServerSideProps<Props> = async (context) => {
  const { params } = context;
  const productId = params?.id as string;

  try {
    // Parallel data fetching
    const [productRes, relatedRes] = await Promise.all([
      fetch(`https://api.example.com/products/${productId}`),
      fetch(`https://api.example.com/products/${productId}/related`)
    ]);

    const [product, relatedProducts] = await Promise.all([
      productRes.json(),
      relatedRes.json()
    ]);

    return {
      props: {
        product,
        relatedProducts
      }
    };
  } catch (error) {
    // Return 404 if product not found
    return {
      notFound: true
    };
  }
};

export default function ProductDetailPage({ product, relatedProducts }: Props) {
  if (!product) {
    return <div>Product not found</div>;
  }

  return (
    <div className="product-detail">
      <div className="product-images">
        {product.images.map((image, index) => (
          <img key={index} src={image} alt={product.name} />
        ))}
      </div>

      <div className="product-info">
        <h1>{product.name}</h1>
        <p className="price">${product.price}</p>
        <p className="description">{product.description}</p>
        <button>Add to Cart</button>
      </div>

      <div className="related-products">
        <h2>Related Products</h2>
        <div className="products-grid">
          {relatedProducts.map((p) => (
            <ProductCard key={p.id} product={p} />
          ))}
        </div>
      </div>
    </div>
  );
}
```

### Authentication with SSR

```typescript
// pages/dashboard.tsx
import { GetServerSideProps } from 'next';
import { getSession } from 'next-auth/react';

interface User {
  id: string;
  name: string;
  email: string;
}

interface DashboardData {
  stats: {
    orders: number;
    revenue: number;
  };
  recentOrders: Order[];
}

interface Props {
  user: User;
  data: DashboardData;
}

export const getServerSideProps: GetServerSideProps<Props> = async (context) => {
  // Check authentication on server
  const session = await getSession(context);

  if (!session) {
    // Redirect to login if not authenticated
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    };
  }

  // Fetch user-specific data
  const response = await fetch('https://api.example.com/dashboard', {
    headers: {
      Authorization: `Bearer ${session.accessToken}`
    }
  });

  const data = await response.json();

  return {
    props: {
      user: session.user,
      data
    }
  };
};

export default function DashboardPage({ user, data }: Props) {
  return (
    <div className="dashboard">
      <h1>Welcome, {user.name}</h1>
      
      <div className="stats">
        <div className="stat-card">
          <h3>Total Orders</h3>
          <p>{data.stats.orders}</p>
        </div>
        <div className="stat-card">
          <h3>Total Revenue</h3>
          <p>${data.stats.revenue}</p>
        </div>
      </div>

      <div className="recent-orders">
        <h2>Recent Orders</h2>
        <OrdersList orders={data.recentOrders} />
      </div>
    </div>
  );
}
```

## Angular Universal SSR

### Setting Up Angular Universal

```typescript
// server.ts
import 'zone.js/node';
import { ngExpressEngine } from '@nguniversal/express-engine';
import * as express from 'express';
import { join } from 'path';
import { AppServerModule } from './src/main.server';

const app = express();
const PORT = process.env.PORT || 4000;
const DIST_FOLDER = join(process.cwd(), 'dist/browser');

// Express engine for rendering Angular
app.engine('html', ngExpressEngine({
  bootstrap: AppServerModule,
}));

app.set('view engine', 'html');
app.set('views', DIST_FOLDER);

// Serve static files
app.get('*.*', express.static(DIST_FOLDER, {
  maxAge: '1y'
}));

// All regular routes use SSR
app.get('*', (req, res) => {
  res.render('index', {
    req,
    providers: [
      { provide: 'REQUEST', useValue: req },
      { provide: 'RESPONSE', useValue: res }
    ]
  });
});

app.listen(PORT, () => {
  console.log(`Node Express server listening on http://localhost:${PORT}`);
});
```

### Angular Component with SSR

```typescript
// app.component.ts
import { Component, OnInit, Inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { TransferState, makeStateKey } from '@angular/platform-browser';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Component({
  selector: 'app-root',
  template: `
    <div class="products">
      <h1>Products</h1>
      
      <div *ngIf="loading">Loading...</div>
      
      <div *ngIf="!loading" class="products-grid">
        <div *ngFor="let product of products" class="product-card">
          <h2>{{ product.name }}</h2>
          <p>\${{ product.price }}</p>
        </div>
      </div>
    </div>
  `
})
export class AppComponent implements OnInit {
  products: Product[] = [];
  loading = true;

  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    private http: HttpClient,
    private transferState: TransferState
  ) {}

  ngOnInit() {
    // Check if running on server or browser
    if (isPlatformServer(this.platformId)) {
      // Running on server - fetch data
      this.fetchProducts().subscribe(products => {
        this.products = products;
        this.loading = false;
        
        // Store data for transfer to browser
        this.transferState.set(PRODUCTS_KEY, products);
      });
    } else {
      // Running on browser - check if data was transferred
      const cachedProducts = this.transferState.get(PRODUCTS_KEY, null);
      
      if (cachedProducts) {
        // Use transferred data
        this.products = cachedProducts;
        this.loading = false;
        
        // Remove from transfer state
        this.transferState.remove(PRODUCTS_KEY);
      } else {
        // Fetch fresh data
        this.fetchProducts().subscribe(products => {
          this.products = products;
          this.loading = false;
        });
      }
    }
  }

  private fetchProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('https://api.example.com/products');
  }
}
```

## Data Fetching Strategies

### Parallel Data Fetching

```typescript
// Next.js - Fetch multiple data sources in parallel
export const getServerSideProps: GetServerSideProps = async () => {
  try {
    // Fetch all data in parallel
    const [productsRes, categoriesRes, settingsRes] = await Promise.all([
      fetch('https://api.example.com/products'),
      fetch('https://api.example.com/categories'),
      fetch('https://api.example.com/settings')
    ]);

    const [products, categories, settings] = await Promise.all([
      productsRes.json(),
      categoriesRes.json(),
      settingsRes.json()
    ]);

    return {
      props: {
        products,
        categories,
        settings
      }
    };
  } catch (error) {
    return {
      props: {
        products: [],
        categories: [],
        settings: {}
      }
    };
  }
};
```

### Error Handling in SSR

```typescript
// Next.js error handling
export const getServerSideProps: GetServerSideProps = async (context) => {
  try {
    const response = await fetch('https://api.example.com/products');
    
    if (!response.ok) {
      if (response.status === 404) {
        return {
          notFound: true
        };
      }

      throw new Error('Failed to fetch products');
    }

    const products = await response.json();

    return {
      props: { products }
    };
  } catch (error) {
    console.error('SSR Error:', error);

    // Redirect to error page
    return {
      redirect: {
        destination: '/error',
        permanent: false
      }
    };

    // Or return error props
    return {
      props: {
        error: 'Failed to load products',
        products: []
      }
    };
  }
};
```

## Hydration

### Understanding Hydration

```typescript
// What happens during hydration:

// 1. Server sends rendered HTML
<div id="root">
  <button onclick="handleClick()">Click me</button>
  <p>Count: 0</p>
</div>

// 2. Browser displays HTML (content visible, not interactive)

// 3. JavaScript downloads and executes

// 4. React/Angular attaches event listeners (hydration)
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
      <p>Count: {count}</p>
    </div>
  );
}

// 5. App becomes interactive
```

### Hydration Mismatch Issues

```typescript
// ❌ BAD: Hydration mismatch
export default function Page({ timestamp }: { timestamp: string }) {
  return (
    <div>
      {/* Server generates one time, client generates different time */}
      <p>Current time: {new Date().toISOString()}</p>
    </div>
  );
}
// Error: Hydration mismatch!

// ✅ GOOD: Use passed timestamp
export default function Page({ timestamp }: { timestamp: string }) {
  return (
    <div>
      <p>Current time: {timestamp}</p>
    </div>
  );
}

// ✅ GOOD: Suppress hydration warning for client-only content
export default function Page() {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  return (
    <div>
      {/* Only render on client */}
      {mounted && (
        <p suppressHydrationWarning>
          Client time: {new Date().toISOString()}
        </p>
      )}
    </div>
  );
}
```

### Client-Only Components

```typescript
// Next.js - Mark component as client-only
import dynamic from 'next/dynamic';

// Disable SSR for specific component
const ClientOnlyMap = dynamic(
  () => import('../components/Map'),
  { ssr: false }
);

export default function Page() {
  return (
    <div>
      <h1>Location</h1>
      {/* Map only renders on client */}
      <ClientOnlyMap />
    </div>
  );
}

// Or use custom hook
function useHasMounted() {
  const [hasMounted, setHasMounted] = useState(false);

  useEffect(() => {
    setHasMounted(true);
  }, []);

  return hasMounted;
}

function Page() {
  const hasMounted = useHasMounted();

  return (
    <div>
      {hasMounted && <BrowserOnlyComponent />}
    </div>
  );
}
```

## Performance Optimization

### Caching Strategies

```typescript
// Next.js - Cache SSR responses
export const getServerSideProps: GetServerSideProps = async (context) => {
  const { res } = context;

  // Cache for 10 seconds, serve stale for 59 seconds
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  );

  const products = await fetchProducts();

  return {
    props: { products }
  };
};

// Express - Cache with Redis
import { createClient } from 'redis';

const redis = createClient();

app.get('/products', async (req, res) => {
  const cacheKey = 'products:all';

  // Check cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return res.send(cached);
  }

  // Render and cache
  const html = await renderToString(<ProductsPage />);
  await redis.setEx(cacheKey, 60, html); // Cache for 60 seconds
  
  res.send(html);
});
```

### Streaming SSR

```typescript
// Next.js 13+ with React 18 Streaming
import { Suspense } from 'react';

// Async Server Component
async function Products() {
  const products = await fetchProducts();
  
  return (
    <div className="products-grid">
      {products.map(p => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}

// Page streams HTML as it's ready
export default function Page() {
  return (
    <div>
      <h1>Our Products</h1>
      
      {/* Show fallback while Products loads */}
      <Suspense fallback={<ProductsSkeleton />}>
        <Products />
      </Suspense>
    </div>
  );
}
```

## TTFB vs TTI

```
Server-Side Rendering Metrics:

TTFB (Time to First Byte):
├─ Request sent (0ms)
└─ First byte received (200ms) ← TTFB

FCP (First Contentful Paint):
├─ HTML parsed
└─ Content rendered (250ms) ← User sees content

TTI (Time to Interactive):
├─ JavaScript downloads (400ms)
├─ JavaScript executes (550ms)
└─ App interactive (600ms) ← TTI

SSR Advantage:
- Fast TTFB and FCP (content visible quickly)
- Slower TTI (need to download and hydrate JS)

CSR Comparison:
- Slow TTFB (empty HTML)
- Slow FCP (no content until JS executes)
- TTI = FCP (interactive when content appears)
```

## Common Mistakes

### 1. Not Handling Browser-Only APIs

```typescript
// ❌ BAD: Using window on server
export const getServerSideProps: GetServerSideProps = async () => {
  const width = window.innerWidth; // Error: window is not defined
  
  return {
    props: { width }
  };
};

// ✅ GOOD: Check environment
export const getServerSideProps: GetServerSideProps = async () => {
  return {
    props: {}
  };
};

function Page() {
  const [width, setWidth] = useState(0);

  useEffect(() => {
    // Safe to use window here (client-only)
    setWidth(window.innerWidth);
  }, []);

  return <div>Width: {width}px</div>;
}
```

### 2. Fetching Data on Client After SSR

```typescript
// ❌ BAD: Double data fetching
export const getServerSideProps: GetServerSideProps = async () => {
  const products = await fetchProducts(); // Fetched on server
  
  return {
    props: { products }
  };
};

function Page({ products: serverProducts }) {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    // Fetching again on client! Wasteful!
    fetchProducts().then(setProducts);
  }, []);

  return <ProductList products={products} />;
}

// ✅ GOOD: Use server data
export const getServerSideProps: GetServerSideProps = async () => {
  const products = await fetchProducts();
  
  return {
    props: { products }
  };
};

function Page({ products }) {
  // Use server data directly
  return <ProductList products={products} />;
}
```

### 3. Slow Data Fetching

```typescript
// ❌ BAD: Sequential fetching
export const getServerSideProps: GetServerSideProps = async () => {
  const products = await fetchProducts(); // Wait
  const categories = await fetchCategories(); // Wait
  const settings = await fetchSettings(); // Wait
  // Takes 3x longer!
  
  return {
    props: { products, categories, settings }
  };
};

// ✅ GOOD: Parallel fetching
export const getServerSideProps: GetServerSideProps = async () => {
  const [products, categories, settings] = await Promise.all([
    fetchProducts(),
    fetchCategories(),
    fetchSettings()
  ]);
  // Much faster!
  
  return {
    props: { products, categories, settings }
  };
};
```

## Best Practices

### 1. Use SSR Selectively

```typescript
// Not everything needs SSR
// SSR for public, SEO-critical pages
pages/
  ├─ index.tsx              (SSR)
  ├─ products/[id].tsx      (SSR)
  ├─ blog/[slug].tsx        (SSR)
  ├─ dashboard.tsx          (CSR - authenticated)
  └─ admin/                 (CSR - admin panel)
```

### 2. Implement Proper Loading States

```typescript
export default function Page({ initialData }) {
  const [data, setData] = useState(initialData);
  const [isRefreshing, setIsRefreshing] = useState(false);

  const refresh = async () => {
    setIsRefreshing(true);
    const newData = await fetchData();
    setData(newData);
    setIsRefreshing(false);
  };

  return (
    <div>
      {isRefreshing ? <Spinner /> : <Content data={data} />}
      <button onClick={refresh}>Refresh</button>
    </div>
  );
}
```

### 3. Monitor Performance

```typescript
// Track SSR performance
export const getServerSideProps: GetServerSideProps = async () => {
  const startTime = Date.now();

  try {
    const data = await fetchData();
    
    const duration = Date.now() - startTime;
    
    // Log slow requests
    if (duration > 1000) {
      console.warn(`Slow SSR request: ${duration}ms`);
    }

    // Send to monitoring service
    analytics.track('ssr_duration', { duration });

    return {
      props: { data }
    };
  } catch (error) {
    analytics.track('ssr_error', { error });
    throw error;
  }
};
```

## When to Use SSR

### Use SSR When

1. **SEO is Critical**: Public content that needs to be indexed
2. **Fast Initial Load**: Need content visible quickly
3. **Social Sharing**: Proper meta tags for link previews
4. **Dynamic Content**: Frequently changing content
5. **Personalization**: User-specific content

### Don't Use SSR When

1. **Highly Interactive**: Heavy client-side interactions
2. **Real-time Updates**: Frequent data changes (use CSR with websockets)
3. **High Server Load**: Can't scale server rendering
4. **Simple Static Content**: Use SSG instead
5. **Authenticated Only**: Private dashboards (use CSR)

## Interview Questions

### 1. What is Server-Side Rendering and how does it differ from CSR?
**Answer**: SSR generates HTML on the server for each request, sending fully-formed HTML to the browser. This provides fast initial content display (good TTFB/FCP) and better SEO. CSR sends minimal HTML and renders on the client after JS loads. SSR has better initial performance but higher server costs. CSR has slower initial load but reduces server load.

### 2. What is hydration in SSR?
**Answer**: Hydration is the process where JavaScript takes over the server-rendered HTML and attaches event listeners to make it interactive. The server sends HTML (content visible), then JS downloads and "hydrates" the HTML by attaching React/Angular event handlers and state. During hydration, the app transitions from static HTML to fully interactive.

### 3. What causes hydration mismatches?
**Answer**: Mismatches occur when server-rendered HTML differs from client-rendered HTML. Common causes: using browser-only APIs (localStorage, window), generating random data, using current timestamps, conditional rendering based on client state. Solutions: ensure server and client render the same HTML, or use client-only rendering for dynamic content.

### 4. How do you optimize SSR performance?
**Answer**: Parallel data fetching with Promise.all, implement caching (CDN, Redis), use streaming SSR, minimize data fetching in getServerSideProps, implement proper error handling, use React 18 Suspense for progressive rendering, cache at CDN edge, and monitor TTFB metrics. Consider ISR as an alternative to SSR.

### 5. What's the difference between SSR and SSG?
**Answer**: SSR renders on each request (dynamic), SSG renders at build time (static). SSR always has fresh data but slower and more expensive. SSG is fast and cheap but data can be stale. SSR for dynamic/personalized content, SSG for static content. ISR (Incremental Static Regeneration) combines both approaches.

## Key Takeaways

1. **Fast Initial Load**: SSR provides quick Time to First Byte and First Contentful Paint.
2. **Better SEO**: Search engines receive fully-rendered HTML.
3. **Hydration Required**: JavaScript must download and hydrate for interactivity.
4. **Server Cost**: SSR increases server load compared to serving static files.
5. **TTFB vs TTI Trade-off**: Fast content display but slower interactivity.
6. **Parallel Fetching**: Always fetch data in parallel, never sequentially.
7. **Caching Essential**: Implement aggressive caching to reduce server load.
8. **Not Always Best**: Consider SSG/ISR for content that doesn't change often.

## Resources

- [Next.js Server-Side Rendering](https://nextjs.org/docs/basic-features/data-fetching/get-server-side-props)
- [Angular Universal](https://angular.io/guide/universal)
- [React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023)
- [Web.dev - Rendering on the Web](https://web.dev/rendering-on-the-web/)
- [Patterns.dev - Server-Side Rendering](https://www.patterns.dev/posts/server-side-rendering)
- [Vercel - Understanding Next.js Data Fetching](https://vercel.com/docs/concepts/next.js/data-fetching)
