# Next.js Rendering Modes

## Overview

Next.js supports multiple rendering modes, each optimized for different use cases. Understanding when and how to use Static Site Generation (SSG), Server-Side Rendering (SSR), Incremental Static Regeneration (ISR), and Partial Prerendering (PPR) is crucial for building performant applications.

## Table of Contents

1. [Static Site Generation (SSG)](#static-site-generation-ssg)
2. [Server-Side Rendering (SSR)](#server-side-rendering-ssr)
3. [Incremental Static Regeneration (ISR)](#incremental-static-regeneration-isr)
4. [Partial Prerendering (PPR)](#partial-prerendering-ppr)
5. [Dynamic Params Handling](#dynamic-params-handling)
6. [Choosing the Right Mode](#choosing-the-right-mode)
7. [Performance Implications](#performance-implications)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

## Static Site Generation (SSG)

SSG generates HTML at build time. Pages are pre-rendered and served as static files, offering the best performance and lowest cost.

### Basic SSG with generateStaticParams

```typescript
// app/blog/[slug]/page.tsx
interface Post {
  slug: string;
  title: string;
  content: string;
}

// Generate static params at build time
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());
  
  return posts.map((post: Post) => ({
    slug: post.slug,
  }));
}

// This page will be statically generated for each slug
export default async function BlogPost({ 
  params 
}: { 
  params: { slug: string } 
}) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());
  
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

### Forcing Static Rendering

```typescript
// app/about/page.tsx
export const dynamic = 'force-static';

// This page will always be statically generated
export default function About() {
  return (
    <div>
      <h1>About Us</h1>
      <p>This page is always static</p>
    </div>
  );
}
```

### Static with Metadata

```typescript
// app/products/[id]/page.tsx
import { Metadata } from 'next';

interface Product {
  id: string;
  name: string;
  description: string;
  image: string;
}

export async function generateStaticParams() {
  const products = await fetch('https://api.example.com/products')
    .then(res => res.json());
  
  return products.map((product: Product) => ({
    id: product.id,
  }));
}

// Generate metadata for each product
export async function generateMetadata({ 
  params 
}: { 
  params: { id: string } 
}): Promise<Metadata> {
  const product: Product = await fetch(
    `https://api.example.com/products/${params.id}`
  ).then(res => res.json());
  
  return {
    title: product.name,
    description: product.description,
    openGraph: {
      images: [product.image],
    },
  };
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const product: Product = await fetch(
    `https://api.example.com/products/${params.id}`
  ).then(res => res.json());
  
  return (
    <div>
      <h1>{product.name}</h1>
      <img src={product.image} alt={product.name} />
      <p>{product.description}</p>
    </div>
  );
}
```

### Static with Parallel Data Fetching

```typescript
// app/dashboard/page.tsx
export const dynamic = 'force-static';

async function getUser() {
  return fetch('https://api.example.com/user').then(res => res.json());
}

async function getStats() {
  return fetch('https://api.example.com/stats').then(res => res.json());
}

async function getNotifications() {
  return fetch('https://api.example.com/notifications').then(res => res.json());
}

export default async function Dashboard() {
  // Parallel data fetching at build time
  const [user, stats, notifications] = await Promise.all([
    getUser(),
    getStats(),
    getNotifications(),
  ]);
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <div>Total Sales: {stats.sales}</div>
      <div>Notifications: {notifications.length}</div>
    </div>
  );
}
```

## Server-Side Rendering (SSR)

SSR generates HTML on each request. Use when you need fresh data or request-specific information.

### Force Dynamic Rendering

```typescript
// app/user/profile/page.tsx
export const dynamic = 'force-dynamic';

// This page will be rendered on every request
export default async function Profile() {
  const session = await getServerSession();
  const user = await fetch(`https://api.example.com/users/${session.userId}`, {
    cache: 'no-store', // Ensure fresh data
  }).then(res => res.json());
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Last login: {new Date(user.lastLogin).toLocaleString()}</p>
    </div>
  );
}
```

### SSR with Headers and Cookies

```typescript
// app/api/dashboard/page.tsx
import { cookies, headers } from 'next/headers';

export const dynamic = 'force-dynamic';

export default async function Dashboard() {
  const cookieStore = cookies();
  const token = cookieStore.get('auth-token');
  const headersList = headers();
  const userAgent = headersList.get('user-agent');
  
  const data = await fetch('https://api.example.com/dashboard', {
    headers: {
      Authorization: `Bearer ${token?.value}`,
    },
    cache: 'no-store',
  }).then(res => res.json());
  
  return (
    <div>
      <h1>Dashboard</h1>
      <p>User Agent: {userAgent}</p>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  );
}
```

### SSR with Search Params

```typescript
// app/search/page.tsx
export const dynamic = 'force-dynamic';

interface SearchParams {
  q?: string;
  category?: string;
  page?: string;
}

export default async function SearchPage({
  searchParams,
}: {
  searchParams: SearchParams;
}) {
  const query = searchParams.q || '';
  const category = searchParams.category || 'all';
  const page = parseInt(searchParams.page || '1', 10);
  
  const results = await fetch(
    `https://api.example.com/search?q=${query}&category=${category}&page=${page}`,
    { cache: 'no-store' }
  ).then(res => res.json());
  
  return (
    <div>
      <h1>Search Results for "{query}"</h1>
      <p>Category: {category} | Page: {page}</p>
      <ul>
        {results.map((item: any) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Conditional SSR

```typescript
// app/posts/[id]/page.tsx
import { headers } from 'next/headers';

// Dynamically determine rendering mode
export async function generateMetadata({ params }: { params: { id: string } }) {
  const headersList = headers();
  const isBot = headersList.get('user-agent')?.includes('bot');
  
  // Always SSR for bots (for SEO)
  if (isBot) {
    return { robots: 'index, follow' };
  }
  
  return {};
}

export default async function Post({ params }: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`, {
    cache: 'no-store',
  }).then(res => res.json());
  
  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  );
}
```

## Incremental Static Regeneration (ISR)

ISR combines the benefits of SSG and SSR: pages are statically generated but can be updated at runtime.

### Time-Based Revalidation

```typescript
// app/news/page.tsx
export const revalidate = 60; // Revalidate every 60 seconds

async function getNews() {
  const res = await fetch('https://api.example.com/news', {
    next: { revalidate: 60 },
  });
  return res.json();
}

export default async function NewsPage() {
  const news = await getNews();
  
  return (
    <div>
      <h1>Latest News</h1>
      <ul>
        {news.map((article: any) => (
          <li key={article.id}>{article.title}</li>
        ))}
      </ul>
      <p>Updated every 60 seconds</p>
    </div>
  );
}
```

### Per-Request Revalidation

```typescript
// app/blog/[slug]/page.tsx
export const revalidate = 3600; // Revalidate every hour

export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());
  return posts.map((post: any) => ({ slug: post.slug }));
}

async function getPost(slug: string) {
  const res = await fetch(`https://api.example.com/posts/${slug}`, {
    next: { revalidate: 3600, tags: ['blog-posts'] },
  });
  return res.json();
}

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>Published: {new Date(post.publishedAt).toLocaleDateString()}</p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

### On-Demand Revalidation with revalidatePath

```typescript
// app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { path, secret } = await request.json();
  
  // Verify secret to prevent unauthorized revalidation
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ message: 'Invalid secret' }, { status: 401 });
  }
  
  try {
    // Revalidate specific path
    revalidatePath(path);
    return NextResponse.json({ revalidated: true, path });
  } catch (err) {
    return NextResponse.json({ message: 'Error revalidating' }, { status: 500 });
  }
}

// Usage: POST /api/revalidate
// Body: { "path": "/blog/my-post", "secret": "..." }
```

### On-Demand Revalidation with revalidateTag

```typescript
// app/products/[id]/page.tsx
export const revalidate = 86400; // 24 hours default

async function getProduct(id: string) {
  const res = await fetch(`https://api.example.com/products/${id}`, {
    next: { tags: [`product-${id}`, 'products'] },
  });
  return res.json();
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>Price: ${product.price}</p>
      <p>Stock: {product.stock}</p>
    </div>
  );
}

// app/api/revalidate-product/route.ts
import { revalidateTag } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { productId, secret } = await request.json();
  
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ message: 'Invalid secret' }, { status: 401 });
  }
  
  try {
    // Revalidate all pages tagged with this product
    revalidateTag(`product-${productId}`);
    // Or revalidate all product pages
    // revalidateTag('products');
    
    return NextResponse.json({ revalidated: true, productId });
  } catch (err) {
    return NextResponse.json({ message: 'Error revalidating' }, { status: 500 });
  }
}
```

### ISR with Fallback Pages

```typescript
// app/docs/[...slug]/page.tsx
export const dynamicParams = true; // Allow dynamic params not in generateStaticParams
export const revalidate = 3600;

export async function generateStaticParams() {
  // Pre-generate top 100 most popular docs
  const popularDocs = await fetch('https://api.example.com/docs/popular?limit=100')
    .then(res => res.json());
  
  return popularDocs.map((doc: any) => ({
    slug: doc.slug.split('/'),
  }));
}

export default async function DocPage({ params }: { params: { slug: string[] } }) {
  const slugPath = params.slug.join('/');
  
  try {
    const doc = await fetch(`https://api.example.com/docs/${slugPath}`, {
      next: { revalidate: 3600 },
    }).then(res => {
      if (!res.ok) throw new Error('Doc not found');
      return res.json();
    });
    
    return (
      <article>
        <h1>{doc.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: doc.content }} />
      </article>
    );
  } catch (error) {
    return (
      <div>
        <h1>Documentation Not Found</h1>
        <p>The requested documentation page does not exist.</p>
      </div>
    );
  }
}
```

## Partial Prerendering (PPR)

PPR is an experimental feature that combines static and dynamic rendering in the same page. The static shell loads instantly, while dynamic parts stream in.

### Enabling PPR

```typescript
// next.config.js
module.exports = {
  experimental: {
    ppr: true,
  },
};
```

### Basic PPR with Suspense

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

// Static shell
export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <nav>
        <a href="/profile">Profile</a>
        <a href="/settings">Settings</a>
      </nav>
      
      {/* Dynamic hole - rendered on the server */}
      <Suspense fallback={<div>Loading user data...</div>}>
        <UserData />
      </Suspense>
      
      {/* Another dynamic hole */}
      <Suspense fallback={<div>Loading analytics...</div>}>
        <Analytics />
      </Suspense>
      
      {/* Static footer */}
      <footer>© 2024 My App</footer>
    </div>
  );
}

// Dynamic component
async function UserData() {
  const user = await fetch('https://api.example.com/user', {
    cache: 'no-store',
  }).then(res => res.json());
  
  return (
    <div>
      <h2>Welcome, {user.name}</h2>
      <p>Last login: {new Date(user.lastLogin).toLocaleString()}</p>
    </div>
  );
}

// Dynamic component
async function Analytics() {
  const stats = await fetch('https://api.example.com/analytics', {
    cache: 'no-store',
  }).then(res => res.json());
  
  return (
    <div>
      <h3>Your Stats</h3>
      <p>Views: {stats.views}</p>
      <p>Clicks: {stats.clicks}</p>
    </div>
  );
}
```

### PPR with Streaming

```typescript
// app/products/page.tsx
import { Suspense } from 'react';

export default function ProductsPage() {
  return (
    <div>
      {/* Static header */}
      <header>
        <h1>Our Products</h1>
        <p>Browse our extensive catalog</p>
      </header>
      
      {/* Static categories (built at build time) */}
      <aside>
        <Categories />
      </aside>
      
      {/* Dynamic product list (personalized) */}
      <Suspense fallback={<ProductsSkeleton />}>
        <ProductList />
      </Suspense>
    </div>
  );
}

// Static component
async function Categories() {
  const categories = await fetch('https://api.example.com/categories', {
    next: { revalidate: 3600 },
  }).then(res => res.json());
  
  return (
    <ul>
      {categories.map((cat: any) => (
        <li key={cat.id}>{cat.name}</li>
      ))}
    </ul>
  );
}

// Dynamic component (personalized based on user)
async function ProductList() {
  const products = await fetch('https://api.example.com/products/personalized', {
    cache: 'no-store',
  }).then(res => res.json());
  
  return (
    <div>
      {products.map((product: any) => (
        <div key={product.id}>
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}

function ProductsSkeleton() {
  return (
    <div>
      {[...Array(6)].map((_, i) => (
        <div key={i} className="skeleton">
          <div className="skeleton-title" />
          <div className="skeleton-price" />
        </div>
      ))}
    </div>
  );
}
```

### PPR with Client Components

```typescript
// app/blog/page.tsx
import { Suspense } from 'react';
import { SearchBar } from './search-bar';

export default function BlogPage() {
  return (
    <div>
      {/* Static header */}
      <h1>Blog</h1>
      
      {/* Client component (hydrates on client) */}
      <SearchBar />
      
      {/* Dynamic featured posts */}
      <Suspense fallback={<div>Loading featured posts...</div>}>
        <FeaturedPosts />
      </Suspense>
      
      {/* Static recent posts */}
      <Suspense fallback={<div>Loading posts...</div>}>
        <RecentPosts />
      </Suspense>
    </div>
  );
}

// app/blog/search-bar.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';

export function SearchBar() {
  const [query, setQuery] = useState('');
  const router = useRouter();
  
  const handleSearch = (e: React.FormEvent) => {
    e.preventDefault();
    router.push(`/blog/search?q=${query}`);
  };
  
  return (
    <form onSubmit={handleSearch}>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search posts..."
      />
      <button type="submit">Search</button>
    </form>
  );
}

// Dynamic server component
async function FeaturedPosts() {
  const posts = await fetch('https://api.example.com/posts/featured', {
    cache: 'no-store',
  }).then(res => res.json());
  
  return (
    <section>
      <h2>Featured</h2>
      {posts.map((post: any) => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </section>
  );
}

// Static server component
async function RecentPosts() {
  const posts = await fetch('https://api.example.com/posts/recent', {
    next: { revalidate: 600 },
  }).then(res => res.json());
  
  return (
    <section>
      <h2>Recent Posts</h2>
      {posts.map((post: any) => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </section>
  );
}
```

## Dynamic Params Handling

### Controlling Dynamic Params

```typescript
// app/posts/[id]/page.tsx

// false: Only pre-generated params are valid (404 for others)
export const dynamicParams = false;

export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());
  return posts.map((post: any) => ({ id: post.id }));
}

export default async function Post({ params }: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`)
    .then(res => res.json());
  
  return <article><h1>{post.title}</h1></article>;
}
```

### Dynamic Params with ISR

```typescript
// app/products/[category]/[id]/page.tsx
export const dynamicParams = true; // Allow new params (default)
export const revalidate = 3600;

export async function generateStaticParams() {
  // Pre-generate top products in main categories
  const categories = ['electronics', 'clothing', 'books'];
  const params: Array<{ category: string; id: string }> = [];
  
  for (const category of categories) {
    const products = await fetch(
      `https://api.example.com/products?category=${category}&limit=20`
    ).then(res => res.json());
    
    products.forEach((product: any) => {
      params.push({ category, id: product.id });
    });
  }
  
  return params;
}

export default async function ProductPage({ 
  params 
}: { 
  params: { category: string; id: string } 
}) {
  const product = await fetch(
    `https://api.example.com/products/${params.category}/${params.id}`,
    { next: { revalidate: 3600 } }
  ).then(res => res.json());
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>Category: {params.category}</p>
      <p>Price: ${product.price}</p>
    </div>
  );
}
```

## Choosing the Right Mode

### Decision Framework

```typescript
// Use SSG when:
// - Content changes infrequently
// - Same content for all users
// - Can be pre-rendered at build time
// Examples: Marketing pages, documentation, blogs

// app/about/page.tsx
export const dynamic = 'force-static';
export default function About() {
  return <div>About Us</div>;
}

// Use SSR when:
// - Content changes on every request
// - User-specific or request-specific data
// - Need real-time data
// Examples: User dashboards, admin panels, search

// app/dashboard/page.tsx
export const dynamic = 'force-dynamic';
export default async function Dashboard() {
  const user = await getCurrentUser();
  return <div>Welcome, {user.name}</div>;
}

// Use ISR when:
// - Content updates periodically
// - Can tolerate slightly stale data
// - High traffic with changing content
// Examples: E-commerce, news sites, social feeds

// app/news/page.tsx
export const revalidate = 60;
export default async function News() {
  const articles = await fetchNews();
  return <div>{/* articles */}</div>;
}

// Use PPR when:
// - Mix of static and dynamic content
// - Want instant static shell + personalized data
// - Complex pages with varying update frequencies
// Examples: Product pages, hybrid dashboards

// app/product/[id]/page.tsx (with experimental PPR)
export default function Product() {
  return (
    <>
      <StaticHeader />
      <Suspense fallback={<Skeleton />}>
        <DynamicReviews />
      </Suspense>
    </>
  );
}
```

### Real-World Scenarios

```typescript
// E-commerce Product Page (Hybrid Approach)
// app/products/[id]/page.tsx
import { Suspense } from 'react';

export const revalidate = 3600; // ISR for product data

export async function generateStaticParams() {
  const topProducts = await fetch('https://api.example.com/products/top')
    .then(res => res.json());
  return topProducts.map((p: any) => ({ id: p.id }));
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  // Static/ISR product data
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    next: { revalidate: 3600, tags: [`product-${params.id}`] },
  }).then(res => res.json());
  
  return (
    <div>
      {/* Static product info */}
      <h1>{product.name}</h1>
      <img src={product.image} alt={product.name} />
      <p>{product.description}</p>
      <p>${product.price}</p>
      
      {/* Dynamic inventory (SSR) */}
      <Suspense fallback={<div>Checking availability...</div>}>
        <InventoryStatus productId={params.id} />
      </Suspense>
      
      {/* Dynamic personalized recommendations */}
      <Suspense fallback={<div>Loading recommendations...</div>}>
        <Recommendations productId={params.id} />
      </Suspense>
      
      {/* Static/ISR reviews */}
      <Suspense fallback={<div>Loading reviews...</div>}>
        <Reviews productId={params.id} />
      </Suspense>
    </div>
  );
}

async function InventoryStatus({ productId }: { productId: string }) {
  const inventory = await fetch(
    `https://api.example.com/inventory/${productId}`,
    { cache: 'no-store' }
  ).then(res => res.json());
  
  return (
    <div>
      {inventory.inStock ? (
        <span className="text-green-600">In Stock ({inventory.quantity})</span>
      ) : (
        <span className="text-red-600">Out of Stock</span>
      )}
    </div>
  );
}

async function Recommendations({ productId }: { productId: string }) {
  const recs = await fetch(
    `https://api.example.com/recommendations/${productId}`,
    { cache: 'no-store' }
  ).then(res => res.json());
  
  return (
    <div>
      <h2>You Might Also Like</h2>
      {recs.map((rec: any) => (
        <div key={rec.id}>{rec.name}</div>
      ))}
    </div>
  );
}

async function Reviews({ productId }: { productId: string }) {
  const reviews = await fetch(
    `https://api.example.com/reviews/${productId}`,
    { next: { revalidate: 600 } }
  ).then(res => res.json());
  
  return (
    <div>
      <h2>Customer Reviews</h2>
      {reviews.map((review: any) => (
        <div key={review.id}>
          <p>{review.rating} stars</p>
          <p>{review.comment}</p>
        </div>
      ))}
    </div>
  );
}
```

## Performance Implications

### Build Time vs Runtime Performance

```typescript
// SSG: Fast runtime, slow build (for many pages)
// app/posts/[slug]/page.tsx
export async function generateStaticParams() {
  // If you have 10,000 posts, build will take a long time
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());
  return posts.map((p: any) => ({ slug: p.slug }));
}

// Solution: Pre-render popular pages, use ISR for rest
export const dynamicParams = true; // Allow runtime generation
export const revalidate = 3600;

export async function generateStaticParams() {
  // Only pre-render top 100
  const popularPosts = await fetch('https://api.example.com/posts/popular?limit=100')
    .then(res => res.json());
  return popularPosts.map((p: any) => ({ slug: p.slug }));
}
```

### Caching Strategies

```typescript
// app/api/data/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  const data = await fetch('https://api.example.com/data', {
    // No caching (SSR)
    cache: 'no-store',
  }).then(res => res.json());
  
  return NextResponse.json(data);
}

// With revalidation (ISR)
export async function GET() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 },
  }).then(res => res.json());
  
  return NextResponse.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=120',
    },
  });
}

// Force cache (SSG)
export async function GET() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache',
  }).then(res => res.json());
  
  return NextResponse.json(data);
}
```

### Monitoring and Debugging

```typescript
// next.config.js
module.exports = {
  logging: {
    fetches: {
      fullUrl: true,
    },
  },
};

// app/posts/[slug]/page.tsx
export const dynamic = 'error'; // Throw error if page is dynamic

export default async function Post({ params }: { params: { slug: string } }) {
  // This will error if any fetch is not cached
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());
  
  return <article>{post.title}</article>;
}
```

## Common Mistakes

### 1. Not Setting Cache Correctly

```typescript
// WRONG: Expecting static but using no-store
export const dynamic = 'force-static';

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store', // This makes the page dynamic!
  }).then(res => res.json());
  
  return <div>{data.value}</div>;
}

// CORRECT: Use appropriate cache directive
export const dynamic = 'force-static';

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 }, // ISR
  }).then(res => res.json());
  
  return <div>{data.value}</div>;
}
```

### 2. Overusing SSR

```typescript
// WRONG: SSR for rarely changing data
export const dynamic = 'force-dynamic';

export default async function About() {
  const content = await fetch('https://api.example.com/about', {
    cache: 'no-store',
  }).then(res => res.json());
  
  return <div>{content.text}</div>;
}

// CORRECT: Use ISR instead
export const revalidate = 3600;

export default async function About() {
  const content = await fetch('https://api.example.com/about', {
    next: { revalidate: 3600 },
  }).then(res => res.json());
  
  return <div>{content.text}</div>;
}
```

### 3. Missing generateStaticParams

```typescript
// WRONG: Dynamic params without generation strategy
export default async function Post({ params }: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`)
    .then(res => res.json());
  return <article>{post.title}</article>;
}

// CORRECT: Provide generation strategy
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts/popular?limit=100')
    .then(res => res.json());
  return posts.map((p: any) => ({ id: p.id }));
}

export const dynamicParams = true;
export const revalidate = 3600;

export default async function Post({ params }: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`, {
    next: { revalidate: 3600 },
  }).then(res => res.json());
  return <article>{post.title}</article>;
}
```

## Best Practices

### 1. Use the Right Tool for the Job

```typescript
// Marketing/Content Pages: SSG
// app/(marketing)/about/page.tsx
export const dynamic = 'force-static';

// User Dashboards: SSR
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic';

// E-commerce/News: ISR
// app/products/[id]/page.tsx
export const revalidate = 3600;

// Complex Apps: PPR
// app/feed/page.tsx (with PPR enabled)
// Static shell + dynamic personalized content
```

### 2. Optimize Build Times

```typescript
// Pre-render critical paths only
export async function generateStaticParams() {
  const critical = await fetch('https://api.example.com/critical-pages')
    .then(res => res.json());
  return critical.map((p: any) => ({ slug: p.slug }));
}

export const dynamicParams = true;
export const revalidate = 3600;
```

### 3. Use Cache Tags for Granular Revalidation

```typescript
// Tag related data
const product = await fetch(`https://api.example.com/products/${id}`, {
  next: { 
    revalidate: 3600, 
    tags: [`product-${id}`, 'products', `category-${categoryId}`] 
  },
});

// Revalidate specific product
revalidateTag(`product-${id}`);

// Revalidate entire category
revalidateTag(`category-${categoryId}`);
```

### 4. Monitor Rendering Modes

```typescript
// Add headers to identify rendering mode
export default async function Page() {
  return (
    <div>
      <meta name="x-render-mode" content="static" />
      {/* content */}
    </div>
  );
}
```

## Interview Questions

### Q1: What's the difference between SSG, SSR, and ISR?

**Answer:** SSG pre-renders pages at build time (fast, static HTML). SSR renders on each request (fresh data, slower). ISR combines both: pages are pre-rendered but can be updated at runtime based on time or on-demand revalidation.

### Q2: When would you use PPR?

**Answer:** PPR is ideal for pages with both static and dynamic content, like e-commerce product pages (static product info, dynamic inventory/recommendations) or news sites (static layout, dynamic personalized feed).

### Q3: How does revalidateTag differ from revalidatePath?

**Answer:** revalidatePath invalidates a specific route path. revalidateTag invalidates all fetch requests tagged with a specific tag, allowing granular cache invalidation across multiple pages.

### Q4: What happens when dynamicParams is false?

**Answer:** Only URLs with params generated by generateStaticParams are valid. Requests with other params return 404. This prevents runtime generation and ensures all pages are pre-rendered.

### Q5: How do you handle a site with 1 million pages?

**Answer:** Pre-render popular pages with generateStaticParams (e.g., top 1000), enable dynamicParams for runtime generation, use ISR with appropriate revalidation times, and implement on-demand revalidation for critical updates.

## Key Takeaways

1. SSG is fastest but only for static content
2. SSR provides fresh data but is slower
3. ISR combines benefits of both with stale-while-revalidate
4. PPR enables instant static shell with dynamic holes
5. Use generateStaticParams to control which pages are pre-rendered
6. dynamicParams controls whether non-pre-rendered pages are allowed
7. Cache tags enable granular revalidation strategies
8. Choose rendering mode based on data freshness requirements and traffic patterns
9. Monitor build times and optimize with selective pre-rendering
10. Use appropriate cache directives (no-store, revalidate, force-cache)

## Resources

- [Next.js Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [Next.js Rendering](https://nextjs.org/docs/app/building-your-application/rendering)
- [Incremental Static Regeneration](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating)
- [Partial Prerendering](https://nextjs.org/docs/app/building-your-application/rendering/partial-prerendering)
- [generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)
