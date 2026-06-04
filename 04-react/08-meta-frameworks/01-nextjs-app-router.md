# Next.js App Router: File Conventions and Advanced Routing

## Table of Contents
- [Introduction](#introduction)
- [File-based Routing](#file-based-routing)
- [Special Files](#special-files)
- [Parallel Routes](#parallel-routes)
- [Intercepting Routes](#intercepting-routes)
- [Route Groups](#route-groups)
- [Dynamic Routes](#dynamic-routes)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Next.js App Router (introduced in Next.js 13) represents a fundamental shift in how Next.js applications are structured. Built on React Server Components, it introduces new file conventions, routing patterns, and capabilities that enable more sophisticated application architectures.

## File-based Routing

### Directory Structure

```
app/
├── layout.tsx          # Root layout (required)
├── page.tsx            # Home page (/)
├── loading.tsx         # Loading UI
├── error.tsx           # Error UI
├── not-found.tsx       # 404 UI
├── global.css          # Global styles
│
├── blog/
│   ├── layout.tsx      # Blog layout
│   ├── page.tsx        # Blog index (/blog)
│   ├── loading.tsx     # Blog loading
│   ├── error.tsx       # Blog error
│   │
│   └── [slug]/
│       ├── page.tsx    # Blog post (/blog/my-post)
│       └── opengraph-image.tsx  # OG image
│
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx        # /dashboard
│   │
│   ├── settings/
│   │   └── page.tsx    # /dashboard/settings
│   │
│   └── analytics/
│       └── page.tsx    # /dashboard/analytics
│
└── api/
    └── route.ts        # API route handler
```

### Basic Page Component

```typescript
// app/page.tsx
export default function HomePage() {
  return (
    <main>
      <h1>Welcome to Home Page</h1>
      <p>This is the root route (/)</p>
    </main>
  );
}

// app/about/page.tsx
export default function AboutPage() {
  return (
    <main>
      <h1>About Us</h1>
      <p>This renders at /about</p>
    </main>
  );
}

// app/blog/posts/page.tsx
export default function BlogPostsPage() {
  return (
    <main>
      <h1>Blog Posts</h1>
      <p>This renders at /blog/posts</p>
    </main>
  );
}
```

## Special Files

### 1. layout.tsx - Shared UI

Layouts wrap pages and preserve state across navigation.

```typescript
// app/layout.tsx (Root Layout - Required)
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'My App',
  description: 'Created with Next.js'
};

export default function RootLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <header>
          <nav>
            <a href="/">Home</a>
            <a href="/blog">Blog</a>
          </nav>
        </header>
        
        {children}
        
        <footer>
          <p>&copy; 2024 My App</p>
        </footer>
      </body>
    </html>
  );
}

// app/blog/layout.tsx (Nested Layout)
export default function BlogLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="blog-container">
      <aside className="blog-sidebar">
        <h3>Categories</h3>
        <ul>
          <li><a href="/blog/tech">Tech</a></li>
          <li><a href="/blog/business">Business</a></li>
        </ul>
      </aside>
      
      <main className="blog-content">
        {children}
      </main>
    </div>
  );
}

// Layout nesting:
// ┌─────────────────────────────────────┐
// │ Root Layout (app/layout.tsx)        │
// │ ┌─────────────────────────────────┐ │
// │ │ Blog Layout (app/blog/layout)   │ │
// │ │ ┌─────────────────────────────┐ │ │
// │ │ │ Page (app/blog/[slug]/page) │ │ │
// │ │ └─────────────────────────────┘ │ │
// │ └─────────────────────────────────┘ │
// └─────────────────────────────────────┘
```

### 2. page.tsx - Route UI

Defines the UI for a route.

```typescript
// app/products/page.tsx
async function getProducts() {
  const res = await fetch('https://api.example.com/products');
  return res.json();
}

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1>Products</h1>
      <ul>
        {products.map((product: Product) => (
          <li key={product.id}>{product.name}</li>
        ))}
      </ul>
    </div>
  );
}

// With params and searchParams
export default async function ProductPage({
  params,
  searchParams
}: {
  params: { id: string };
  searchParams: { sort?: string; filter?: string };
}) {
  return (
    <div>
      <p>Product ID: {params.id}</p>
      <p>Sort: {searchParams.sort}</p>
      <p>Filter: {searchParams.filter}</p>
    </div>
  );
}
```

### 3. loading.tsx - Loading UI

Automatically wraps page in Suspense with loading as fallback.

```typescript
// app/blog/loading.tsx
export default function Loading() {
  return (
    <div className="loading-skeleton">
      <div className="skeleton-title" />
      <div className="skeleton-content" />
      <div className="skeleton-content" />
      <div className="skeleton-content" />
    </div>
  );
}

// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="dashboard-skeleton">
      <div className="skeleton-card" />
      <div className="skeleton-card" />
      <div className="skeleton-card" />
      <div className="skeleton-chart" />
    </div>
  );
}

// Next.js automatically does:
// <Suspense fallback={<Loading />}>
//   <Page />
// </Suspense>
```

### 4. error.tsx - Error Boundaries

Handles errors in route segments.

```typescript
// app/blog/error.tsx
'use client'; // Error components must be Client Components

export default function Error({
  error,
  reset
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="error-container">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}

// app/dashboard/error.tsx
'use client';

import { useEffect } from 'react';

export default function DashboardError({
  error,
  reset
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to error reporting service
    console.error('Dashboard error:', error);
  }, [error]);

  return (
    <div>
      <h2>Dashboard Error</h2>
      <p>Failed to load dashboard data</p>
      <button onClick={reset}>Retry</button>
    </div>
  );
}
```

### 5. not-found.tsx - 404 UI

Renders when notFound() is called or route doesn't exist.

```typescript
// app/not-found.tsx (Root 404)
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="not-found">
      <h2>404 - Page Not Found</h2>
      <p>Could not find the requested resource</p>
      <Link href="/">Return Home</Link>
    </div>
  );
}

// app/blog/[slug]/not-found.tsx (Segment-specific)
import Link from 'next/link';

export default function PostNotFound() {
  return (
    <div>
      <h2>Blog Post Not Found</h2>
      <p>The post you're looking for doesn't exist</p>
      <Link href="/blog">Back to Blog</Link>
    </div>
  );
}

// Triggering not-found
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation';

async function getPost(slug: string) {
  const post = await db.post.findUnique({ where: { slug } });
  if (!post) {
    notFound(); // Renders not-found.tsx
  }
  return post;
}

export default async function PostPage({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <article>{post.title}</article>;
}
```

### 6. template.tsx - Re-rendered Layout

Similar to layout but re-renders on navigation.

```typescript
// app/template.tsx
export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <div>
      {/* This re-renders on every navigation */}
      {children}
    </div>
  );
}

// Use case: Animation on route change
'use client';

import { motion } from 'framer-motion';

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3 }}
    >
      {children}
    </motion.div>
  );
}
```

## Parallel Routes

Render multiple pages in the same layout simultaneously using `@folder` convention.

### Basic Parallel Routes

```
app/
└── dashboard/
    ├── layout.tsx
    ├── page.tsx
    ├── @analytics/
    │   └── page.tsx
    ├── @team/
    │   └── page.tsx
    └── @activity/
        └── page.tsx
```

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  team,
  activity
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
  activity: React.ReactNode;
}) {
  return (
    <div className="dashboard-grid">
      <div className="main-content">
        {children}
      </div>
      
      <div className="analytics-panel">
        {analytics}
      </div>
      
      <div className="team-panel">
        {team}
      </div>
      
      <div className="activity-panel">
        {activity}
      </div>
    </div>
  );
}

// app/dashboard/@analytics/page.tsx
export default async function AnalyticsSlot() {
  const data = await fetchAnalytics();
  return (
    <div>
      <h2>Analytics</h2>
      <Chart data={data} />
    </div>
  );
}

// app/dashboard/@team/page.tsx
export default async function TeamSlot() {
  const team = await fetchTeam();
  return (
    <div>
      <h2>Team Members</h2>
      <TeamList members={team} />
    </div>
  );
}

// app/dashboard/@activity/page.tsx
export default async function ActivitySlot() {
  const activity = await fetchActivity();
  return (
    <div>
      <h2>Recent Activity</h2>
      <ActivityFeed items={activity} />
    </div>
  );
}
```

### Conditional Parallel Routes

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  team
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  const user = getCurrentUser();

  return (
    <div>
      {children}
      
      {/* Only show analytics to admins */}
      {user.isAdmin && analytics}
      
      {/* Show team to all authenticated users */}
      {user && team}
    </div>
  );
}
```

### Independent Loading States

```typescript
// app/dashboard/@analytics/loading.tsx
export default function AnalyticsLoading() {
  return <AnalyticsSkeleton />;
}

// app/dashboard/@team/loading.tsx
export default function TeamLoading() {
  return <TeamSkeleton />;
}

// Each slot loads independently!
```

## Intercepting Routes

Intercept routes to show modals or overlays without changing URL.

### Convention: (.)folder, (..)folder, (..)(..)folder, (...)folder

```
app/
├── page.tsx
├── feed/
│   ├── page.tsx
│   └── [id]/
│       └── page.tsx        # /feed/123 (full page)
└── @modal/
    ├── (.)feed/
    │   └── [id]/
    │       └── page.tsx    # Intercepts /feed/123 (shows modal)
    └── default.tsx
```

### Modal Implementation

```typescript
// app/layout.tsx
export default function RootLayout({
  children,
  modal
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <html>
      <body>
        {children}
        {modal}
      </body>
    </html>
  );
}

// app/feed/page.tsx
export default function FeedPage() {
  return (
    <div className="feed">
      <Link href="/feed/1">Post 1</Link>
      <Link href="/feed/2">Post 2</Link>
      <Link href="/feed/3">Post 3</Link>
    </div>
  );
}

// app/feed/[id]/page.tsx (Full page)
export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await fetchPost(params.id);
  return (
    <article className="full-page-post">
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

// app/@modal/(.)feed/[id]/page.tsx (Intercepted - Modal)
'use client';

import { useRouter } from 'next/navigation';

export default async function PostModal({ params }: { params: { id: string } }) {
  const router = useRouter();
  const post = await fetchPost(params.id);

  return (
    <div className="modal-overlay" onClick={() => router.back()}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <button onClick={() => router.back()}>Close</button>
        <h2>{post.title}</h2>
        <p>{post.content}</p>
      </div>
    </div>
  );
}

// app/@modal/default.tsx
export default function Default() {
  return null; // No modal by default
}

/*
Behavior:
- Click link on /feed → Shows modal, URL is /feed/123
- Refresh on /feed/123 → Shows full page (interceptor doesn't apply)
- Direct navigation to /feed/123 → Shows full page
*/
```

### Photo Gallery Example

```
app/
├── photos/
│   ├── page.tsx          # Gallery grid
│   └── [id]/
│       └── page.tsx      # Full page photo
└── @modal/
    ├── (.)photos/
    │   └── [id]/
    │       └── page.tsx  # Modal photo
    └── default.tsx
```

```typescript
// app/photos/page.tsx
export default function PhotosPage() {
  return (
    <div className="photo-grid">
      {photos.map(photo => (
        <Link key={photo.id} href={`/photos/${photo.id}`}>
          <img src={photo.thumbnail} alt={photo.title} />
        </Link>
      ))}
    </div>
  );
}

// app/@modal/(.)photos/[id]/page.tsx
'use client';

export default function PhotoModal({ params }: { params: { id: string } }) {
  const router = useRouter();

  return (
    <div className="photo-modal">
      <button onClick={() => router.back()}>×</button>
      <img src={`/photos/${params.id}/full.jpg`} alt="Photo" />
    </div>
  );
}
```

## Route Groups

Organize routes without affecting URL using `(folder)` convention.

### Basic Route Groups

```
app/
├── (marketing)/
│   ├── layout.tsx        # Marketing layout
│   ├── page.tsx          # / (home)
│   ├── about/
│   │   └── page.tsx      # /about
│   └── contact/
│       └── page.tsx      # /contact
│
├── (shop)/
│   ├── layout.tsx        # Shop layout
│   ├── products/
│   │   └── page.tsx      # /products
│   └── cart/
│       └── page.tsx      # /cart
│
└── (dashboard)/
    ├── layout.tsx        # Dashboard layout
    ├── dashboard/
    │   └── page.tsx      # /dashboard
    └── settings/
        └── page.tsx      # /settings
```

```typescript
// app/(marketing)/layout.tsx
export default function MarketingLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="marketing-layout">
      <nav className="marketing-nav">
        <Link href="/">Home</Link>
        <Link href="/about">About</Link>
        <Link href="/contact">Contact</Link>
      </nav>
      {children}
    </div>
  );
}

// app/(shop)/layout.tsx
export default function ShopLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="shop-layout">
      <nav className="shop-nav">
        <Link href="/products">Products</Link>
        <Link href="/cart">Cart</Link>
      </nav>
      {children}
    </div>
  );
}

// app/(dashboard)/layout.tsx
export default function DashboardLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard-layout">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}
```

### Multiple Root Layouts

```
app/
├── (public)/
│   ├── layout.tsx        # Public layout
│   ├── page.tsx
│   └── about/
│       └── page.tsx
│
└── (authenticated)/
    ├── layout.tsx        # Authenticated layout
    ├── dashboard/
    │   └── page.tsx
    └── profile/
        └── page.tsx
```

```typescript
// app/(public)/layout.tsx
export default function PublicLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <PublicHeader />
        {children}
        <PublicFooter />
      </body>
    </html>
  );
}

// app/(authenticated)/layout.tsx
export default function AuthenticatedLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <AuthenticatedHeader />
        <Sidebar />
        <main>{children}</main>
      </body>
    </html>
  );
}
```

### Organization by Feature

```
app/
├── (auth)/
│   ├── login/
│   ├── register/
│   └── forgot-password/
│
├── (blog)/
│   ├── posts/
│   ├── categories/
│   └── authors/
│
└── (ecommerce)/
    ├── products/
    ├── checkout/
    └── orders/
```

## Dynamic Routes

### Basic Dynamic Route

```typescript
// app/blog/[slug]/page.tsx
export default async function BlogPost({
  params
}: {
  params: { slug: string };
}) {
  const post = await getPost(params.slug);
  return <article>{post.title}</article>;
}

// app/products/[id]/page.tsx
export default async function Product({
  params
}: {
  params: { id: string };
}) {
  const product = await getProduct(params.id);
  return <div>{product.name}</div>;
}
```

### Catch-all Routes

```typescript
// app/docs/[...slug]/page.tsx
// Matches: /docs/a, /docs/a/b, /docs/a/b/c
export default function Docs({
  params
}: {
  params: { slug: string[] };
}) {
  return (
    <div>
      <p>Path: {params.slug.join('/')}</p>
    </div>
  );
}

// app/shop/[[...slug]]/page.tsx
// Matches: /shop, /shop/a, /shop/a/b (optional catch-all)
export default function Shop({
  params
}: {
  params: { slug?: string[] };
}) {
  const path = params.slug?.join('/') || 'home';
  return <div>Shop: {path}</div>;
}
```

### generateStaticParams

```typescript
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getPosts();
  
  return posts.map((post) => ({
    slug: post.slug
  }));
}

export default async function Post({
  params
}: {
  params: { slug: string };
}) {
  const post = await getPost(params.slug);
  return <article>{post.title}</article>;
}

// Multiple dynamic segments
// app/blog/[category]/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getAllPosts();
  
  return posts.map((post) => ({
    category: post.category,
    slug: post.slug
  }));
}
```

## Common Mistakes

### 1. Not Understanding Layout Persistence

```typescript
// ❌ BAD: Putting page-specific code in layout
// app/blog/layout.tsx
export default function BlogLayout({
  children
}: {
  children: React.ReactNode;
}) {
  const [selectedCategory, setSelectedCategory] = useState('all');
  
  // This state persists across page navigations!
  // Might cause unexpected behavior
  
  return <div>{children}</div>;
}

// ✅ GOOD: Keep layout minimal, page-specific state in pages
// app/blog/layout.tsx
export default function BlogLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <BlogNav />
      {children}
    </div>
  );
}
```

### 2. Missing default.tsx for Parallel Routes

```typescript
// ❌ BAD: No default.tsx
// Will show 404 for non-matching routes

// ✅ GOOD: Always provide default.tsx
// app/@modal/default.tsx
export default function Default() {
  return null;
}
```

### 3. Incorrect Intercepting Route Levels

```typescript
// ❌ WRONG: Using wrong number of (..)
// app/feed/@modal/(.)post/[id]/page.tsx
// Doesn't intercept /feed/post/123 correctly

// ✅ CORRECT: Match the directory structure
// app/feed/@modal/(.)post/[id]/page.tsx
// Correctly intercepts /feed/post/123
```

## Best Practices

### 1. Organize with Route Groups

```
app/
├── (auth)/
│   ├── login/
│   └── register/
├── (marketing)/
│   ├── page.tsx
│   └── about/
└── (app)/
    ├── dashboard/
    └── settings/
```

### 2. Use Parallel Routes for Complex UIs

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  metrics,
  activity,
  notifications
}: {
  children: React.ReactNode;
  metrics: React.ReactNode;
  activity: React.ReactNode;
  notifications: React.ReactNode;
}) {
  return (
    <div className="dashboard-grid">
      <div>{metrics}</div>
      <div>{children}</div>
      <div>{activity}</div>
      <div>{notifications}</div>
    </div>
  );
}
```

### 3. Modal Patterns with Intercepting Routes

```typescript
// Use for: Photo galleries, quick views, forms
// Provides: Soft navigation with shareable URLs
```

## Interview Questions

### Q1: What's the difference between layout.tsx and template.tsx?

**Answer:** Layouts persist across navigations and maintain state (like scroll position, input values). Templates re-render on every navigation. Use layouts for most cases, templates only when you need to reset state or trigger animations on navigation.

### Q2: How do parallel routes differ from nested routes?

**Answer:** Parallel routes render multiple page sections simultaneously in the same layout using the @folder convention. Nested routes create a hierarchy where children render inside parents. Parallel routes allow independent loading states and error handling for different sections.

### Q3: What are intercepting routes used for?

**Answer:** Intercepting routes allow showing content in a modal or overlay when navigating via links, while showing a full page on direct navigation or refresh. Common use cases include photo galleries, quick views, and authentication modals. They use the (.)folder convention to match routes.

### Q4: Why use route groups?

**Answer:** Route groups (folder) organize code without affecting URLs, allow multiple root layouts, and help structure large applications by feature or authentication state. They don't appear in the URL path.

### Q5: How does generateStaticParams work?

**Answer:** generateStaticParams defines which dynamic route segments to pre-render at build time. It returns an array of parameter objects. Next.js generates static pages for each combination. For paths not returned, behavior depends on dynamicParams config (404 or generate on-demand).

## Key Takeaways

1. **File conventions are semantic** - Each special file has a specific purpose
2. **Layouts are persistent** - State preserved across navigation
3. **Parallel routes for complex UIs** - Multiple sections with independent loading/errors
4. **Intercepting routes for modals** - Soft navigation with shareable URLs
5. **Route groups for organization** - Don't affect URLs, enable multiple layouts
6. **loading.tsx wraps in Suspense** - Automatic loading states
7. **error.tsx creates error boundaries** - Automatic error handling
8. **Templates re-render** - Use for animations or resetting state

## Resources

### Official Documentation
- [App Router Documentation](https://nextjs.org/docs/app)
- [Routing Fundamentals](https://nextjs.org/docs/app/building-your-application/routing)
- [Parallel Routes](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes)
- [Intercepting Routes](https://nextjs.org/docs/app/building-your-application/routing/intercepting-routes)
- [Route Groups](https://nextjs.org/docs/app/building-your-application/routing/route-groups)

### Examples
- [Next.js Examples Repository](https://github.com/vercel/next.js/tree/canary/examples)
- [App Router Playground](https://github.com/vercel/app-playground)

---

*Last Updated: 2026-05 - Next.js 15 current*
