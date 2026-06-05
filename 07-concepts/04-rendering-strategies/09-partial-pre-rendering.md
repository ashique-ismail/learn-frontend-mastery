# Partial Pre-Rendering (PPR)

## The Idea

**In plain English:** Partial Pre-Rendering (PPR) is a way to build web pages where some parts are prepared in advance (like a printed menu at a restaurant) and other parts are filled in fresh for each person who visits (like writing your name on a name tag). The pre-made parts load instantly, while the personalized parts get added a moment later.

**Real-world analogy:** Think of a movie theater lobby where most of the space is set up the same way for every showing — the seats, signs, concession stand layout, and screen are all fixed in place before anyone arrives. But the "Now Showing" board behind the ticket counter gets updated for each specific screening with your movie title and showtime.

- The pre-arranged seats and signs = static shell (built ahead of time, same for everyone)
- The "Now Showing" board = dynamic holes (filled in fresh for each request)
- The ticket counter worker updating the board = the server generating personalized content at request time

---

## Overview

Partial Pre-Rendering (PPR) is a cutting-edge rendering strategy introduced in Next.js 14+ that combines the benefits of Static Site Generation (SSG) and Server-Side Rendering (SSR) in a single page. PPR allows you to pre-render the static shell of a page at build time while deferring dynamic content to request time, creating an optimal balance between performance and dynamic content delivery.

## How Partial Pre-Rendering Works

```
Traditional Approaches:

SSG (All Static):
┌─────────────────────────────────────┐
│  Build Time: Generate everything    │
│  ✓ Fast delivery                    │
│  ✗ Stale content                    │
│  ✗ No personalization               │
└─────────────────────────────────────┘

SSR (All Dynamic):
┌─────────────────────────────────────┐
│  Request Time: Generate everything  │
│  ✓ Fresh content                    │
│  ✓ Personalization                  │
│  ✗ Slow TTFB                        │
│  ✗ Server load                      │
└─────────────────────────────────────┘

Partial Pre-Rendering (PPR):
┌─────────────────────────────────────┐
│  Build Time: Static shell           │
│  + Request Time: Dynamic holes      │
│                                      │
│  ┌───────────────────────────────┐ │
│  │ Static Shell (Build Time)     │ │
│  │ ┌─────────────────────────┐   │ │
│  │ │ Header (Static)         │   │ │
│  │ ├─────────────────────────┤   │ │
│  │ │ Navigation (Static)     │   │ │
│  │ ├─────────────────────────┤   │ │
│  │ │ 🕳️ User Widget (Dynamic) │   │ │
│  │ ├─────────────────────────┤   │ │
│  │ │ Content (Static)        │   │ │
│  │ ├─────────────────────────┤   │ │
│  │ │ 🕳️ Recommendations (Dyn) │   │ │
│  │ ├─────────────────────────┤   │ │
│  │ │ Footer (Static)         │   │ │
│  │ └─────────────────────────┘   │ │
│  └───────────────────────────────┘ │
│                                      │
│  ✓ Fast TTFB (static shell)        │
│  ✓ Fresh dynamic content           │
│  ✓ Personalization                 │
│  ✓ Reduced server load             │
└─────────────────────────────────────┘

Request Flow:
┌──────────┐     ┌─────────┐     ┌──────────────┐
│  Client  │────▶│   CDN   │────▶│ Static Shell │ (Instant!)
└──────────┘     └─────────┘     └──────────────┘
     │                                   │
     │           ┌──────────────────────┘
     │           │
     │           ▼
     │      ┌─────────────┐
     │      │   Server    │
     │      │  Generates  │
     │      │  Dynamic    │
     │      │   Holes     │
     │      └──────┬──────┘
     │             │
     │◀────────────┘
     │   (Stream dynamic content)
```

## Next.js PPR Implementation

### Enabling PPR

```typescript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    ppr: true, // Enable Partial Pre-Rendering
  },
};

module.exports = nextConfig;
```

### Basic PPR Page

```typescript
// app/page.tsx
import { Suspense } from 'react';
import { StaticHeader } from './StaticHeader';
import { UserProfile } from './UserProfile';
import { ProductList } from './ProductList';
import { Recommendations } from './Recommendations';

// This page uses PPR automatically
export default function HomePage() {
  return (
    <div>
      {/* Static: Pre-rendered at build time */}
      <StaticHeader />

      <main>
        {/* Static: Marketing content */}
        <section className="hero">
          <h1>Welcome to Our Store</h1>
          <p>Discover amazing products</p>
        </section>

        {/* Dynamic: User-specific content */}
        <Suspense fallback={<UserSkeleton />}>
          <UserProfile />
        </Suspense>

        {/* Static: Product catalog */}
        <section>
          <h2>Featured Products</h2>
          <ProductList />
        </section>

        {/* Dynamic: Personalized recommendations */}
        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations />
        </Suspense>
      </main>

      {/* Static: Footer */}
      <footer>
        <p>&copy; 2024 Our Store</p>
      </footer>
    </div>
  );
}

// Static component - rendered at build time
function StaticHeader() {
  return (
    <header>
      <nav>
        <a href="/">Home</a>
        <a href="/products">Products</a>
        <a href="/about">About</a>
      </nav>
    </header>
  );
}

// Dynamic component - rendered at request time
async function UserProfile() {
  // This runs on the server at request time
  const user = await fetchCurrentUser();

  return (
    <div className="user-profile">
      <img src={user.avatar} alt={user.name} />
      <p>Welcome back, {user.name}!</p>
    </div>
  );
}

// Static component
function ProductList() {
  // This could fetch from CMS at build time
  const products = [
    { id: 1, name: 'Product 1', price: 29.99 },
    { id: 2, name: 'Product 2', price: 39.99 },
  ];

  return (
    <div className="product-grid">
      {products.map((product) => (
        <div key={product.id}>
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}

// Dynamic component - personalized
async function Recommendations() {
  const user = await fetchCurrentUser();
  const recommendations = await fetchRecommendations(user.id);

  return (
    <div className="recommendations">
      <h2>Recommended for You</h2>
      {recommendations.map((item) => (
        <div key={item.id}>
          <h3>{item.name}</h3>
          <p>{item.description}</p>
        </div>
      ))}
    </div>
  );
}
```

### E-commerce Product Page with PPR

```typescript
// app/products/[id]/page.tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  images: string[];
}

interface PageProps {
  params: { id: string };
}

export default function ProductPage({ params }: PageProps) {
  return (
    <div className="product-page">
      {/* Static: Product details (pre-rendered) */}
      <Suspense fallback={<ProductSkeleton />}>
        <ProductDetails productId={params.id} />
      </Suspense>

      <div className="product-actions">
        {/* Dynamic: User-specific pricing/availability */}
        <Suspense fallback={<PricingSkeleton />}>
          <UserPricing productId={params.id} />
        </Suspense>

        {/* Dynamic: Cart actions */}
        <Suspense fallback={<div>Loading...</div>}>
          <AddToCartButton productId={params.id} />
        </Suspense>
      </div>

      {/* Static: Product reviews */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews productId={params.id} />
      </Suspense>

      {/* Dynamic: Personalized recommendations */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <RelatedProducts productId={params.id} />
      </Suspense>
    </div>
  );
}

// Static component - can be pre-rendered
async function ProductDetails({ productId }: { productId: string }) {
  const product = await fetchProduct(productId);

  if (!product) {
    notFound();
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <div className="images">
        {product.images.map((img, i) => (
          <img key={i} src={img} alt={product.name} />
        ))}
      </div>
      <p>{product.description}</p>
    </div>
  );
}

// Dynamic component - user-specific
async function UserPricing({ productId }: { productId: string }) {
  const user = await fetchCurrentUser();
  const pricing = await fetchPricingForUser(productId, user.id);

  return (
    <div className="pricing">
      <p className="price">${pricing.price}</p>
      {pricing.discount && (
        <p className="discount">Save {pricing.discount}%</p>
      )}
      <p className="stock">
        {pricing.inStock ? 'In Stock' : 'Out of Stock'}
      </p>
    </div>
  );
}

// Dynamic component - requires user session
async function AddToCartButton({ productId }: { productId: string }) {
  const user = await fetchCurrentUser();
  const inCart = await isProductInCart(productId, user.id);

  return (
    <form action="/api/cart/add" method="POST">
      <input type="hidden" name="productId" value={productId} />
      <button type="submit">
        {inCart ? 'Update Quantity' : 'Add to Cart'}
      </button>
    </form>
  );
}

// Static component - can be cached
async function ProductReviews({ productId }: { productId: string }) {
  const reviews = await fetchReviews(productId);

  return (
    <div className="reviews">
      <h2>Customer Reviews</h2>
      {reviews.map((review) => (
        <div key={review.id}>
          <p>Rating: {review.rating}/5</p>
          <p>{review.comment}</p>
          <p>- {review.author}</p>
        </div>
      ))}
    </div>
  );
}

// Dynamic component - personalized
async function RelatedProducts({ productId }: { productId: string }) {
  const user = await fetchCurrentUser();
  const related = await fetchRelatedProducts(productId, user.id);

  return (
    <div className="related">
      <h2>You Might Also Like</h2>
      {related.map((product) => (
        <div key={product.id}>
          <img src={product.image} alt={product.name} />
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}

async function fetchProduct(id: string): Promise<Product | null> {
  // Fetch from database/CMS
  // This can be cached or pre-rendered
  const response = await fetch(`https://api.example.com/products/${id}`, {
    next: { revalidate: 3600 }, // Cache for 1 hour
  });

  if (!response.ok) return null;
  return response.json();
}

async function fetchCurrentUser() {
  // Fetch current user from session
  // This is dynamic and user-specific
  const { cookies } = await import('next/headers');
  const sessionToken = cookies().get('session')?.value;
  
  const response = await fetch('https://api.example.com/user', {
    headers: { Authorization: `Bearer ${sessionToken}` },
    cache: 'no-store', // Never cache user data
  });

  return response.json();
}
```

### Dashboard with PPR

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div className="dashboard">
      {/* Static: Dashboard layout and navigation */}
      <aside className="sidebar">
        <nav>
          <a href="/dashboard">Overview</a>
          <a href="/dashboard/analytics">Analytics</a>
          <a href="/dashboard/settings">Settings</a>
        </nav>
      </aside>

      <main>
        <h1>Dashboard</h1>

        {/* Dynamic: User info */}
        <Suspense fallback={<UserInfoSkeleton />}>
          <UserInfo />
        </Suspense>

        <div className="stats-grid">
          {/* Dynamic: Live statistics */}
          <Suspense fallback={<StatCard title="Views" loading />}>
            <ViewsCard />
          </Suspense>

          <Suspense fallback={<StatCard title="Revenue" loading />}>
            <RevenueCard />
          </Suspense>

          <Suspense fallback={<StatCard title="Users" loading />}>
            <UsersCard />
          </Suspense>

          <Suspense fallback={<StatCard title="Orders" loading />}>
            <OrdersCard />
          </Suspense>
        </div>

        {/* Dynamic: Recent activity */}
        <Suspense fallback={<ActivitySkeleton />}>
          <RecentActivity />
        </Suspense>

        {/* Static: Help section */}
        <section className="help">
          <h2>Need Help?</h2>
          <p>Check out our documentation or contact support.</p>
        </section>
      </main>
    </div>
  );
}

async function UserInfo() {
  const user = await fetchCurrentUser();

  return (
    <div className="user-info">
      <img src={user.avatar} alt={user.name} />
      <div>
        <h2>{user.name}</h2>
        <p>{user.email}</p>
        <p>Member since {new Date(user.joinedAt).toLocaleDateString()}</p>
      </div>
    </div>
  );
}

async function ViewsCard() {
  const user = await fetchCurrentUser();
  const stats = await fetchViewStats(user.id);

  return (
    <StatCard title="Views" value={stats.views} change={stats.change} />
  );
}

async function RevenueCard() {
  const user = await fetchCurrentUser();
  const stats = await fetchRevenueStats(user.id);

  return (
    <StatCard
      title="Revenue"
      value={`$${stats.revenue}`}
      change={stats.change}
    />
  );
}

async function UsersCard() {
  const user = await fetchCurrentUser();
  const stats = await fetchUserStats(user.id);

  return (
    <StatCard title="Users" value={stats.users} change={stats.change} />
  );
}

async function OrdersCard() {
  const user = await fetchCurrentUser();
  const stats = await fetchOrderStats(user.id);

  return (
    <StatCard title="Orders" value={stats.orders} change={stats.change} />
  );
}

function StatCard({
  title,
  value,
  change,
  loading = false,
}: {
  title: string;
  value?: string | number;
  change?: number;
  loading?: boolean;
}) {
  if (loading) {
    return (
      <div className="stat-card">
        <h3>{title}</h3>
        <div className="skeleton">Loading...</div>
      </div>
    );
  }

  return (
    <div className="stat-card">
      <h3>{title}</h3>
      <p className="value">{value}</p>
      {change !== undefined && (
        <p className={change >= 0 ? 'positive' : 'negative'}>
          {change >= 0 ? '+' : ''}
          {change}%
        </p>
      )}
    </div>
  );
}

async function RecentActivity() {
  const user = await fetchCurrentUser();
  const activities = await fetchRecentActivity(user.id);

  return (
    <div className="activity">
      <h2>Recent Activity</h2>
      {activities.map((activity) => (
        <div key={activity.id} className="activity-item">
          <p>{activity.description}</p>
          <time>{new Date(activity.timestamp).toLocaleString()}</time>
        </div>
      ))}
    </div>
  );
}
```

## Advanced PPR Patterns

### Nested Suspense with Different Priorities

```typescript
// app/complex-page/page.tsx
export default function ComplexPage() {
  return (
    <div>
      {/* Level 1: Critical content */}
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>

      <main>
        {/* Level 2: Important content with nested dynamic parts */}
        <Suspense fallback={<ContentSkeleton />}>
          <MainContent>
            {/* Level 3: Nested dynamic sections */}
            <Suspense fallback={<WidgetSkeleton />}>
              <DynamicWidget />
            </Suspense>
          </MainContent>
        </Suspense>

        {/* Level 2: Secondary content */}
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar>
            {/* Level 3: Nested personalization */}
            <Suspense fallback={<UserCardSkeleton />}>
              <UserCard />
            </Suspense>

            <Suspense fallback={<RecommendationsSkeleton />}>
              <PersonalizedLinks />
            </Suspense>
          </Sidebar>
        </Suspense>
      </main>

      {/* Static footer - no suspense needed */}
      <Footer />
    </div>
  );
}
```

### Conditional PPR Based on Authentication

```typescript
// app/profile/page.tsx
import { Suspense } from 'react';
import { redirect } from 'next/navigation';

export default async function ProfilePage() {
  // Check auth at the edge - still fast!
  const session = await getSession();

  if (!session) {
    redirect('/login');
  }

  return (
    <div>
      {/* Static: Profile layout */}
      <h1>Your Profile</h1>

      {/* Dynamic: User data */}
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile userId={session.userId} />
      </Suspense>

      {/* Dynamic: User posts */}
      <Suspense fallback={<PostsSkeleton />}>
        <UserPosts userId={session.userId} />
      </Suspense>

      {/* Dynamic: Account settings */}
      <Suspense fallback={<SettingsSkeleton />}>
        <AccountSettings userId={session.userId} />
      </Suspense>
    </div>
  );
}
```

### PPR with Streaming Data

```typescript
// app/feed/page.tsx
import { Suspense } from 'react';

export default function FeedPage() {
  return (
    <div>
      <h1>Your Feed</h1>

      {/* Stream posts as they load */}
      <Suspense fallback={<FeedSkeleton />}>
        <FeedContent />
      </Suspense>
    </div>
  );
}

async function FeedContent() {
  const user = await fetchCurrentUser();
  const posts = await fetchFeedPosts(user.id);

  return (
    <div className="feed">
      {posts.map((post) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>

          {/* Nested suspense for post-specific dynamic content */}
          <Suspense fallback={<InteractionsSkeleton />}>
            <PostInteractions postId={post.id} userId={user.id} />
          </Suspense>
        </article>
      ))}
    </div>
  );
}

async function PostInteractions({
  postId,
  userId,
}: {
  postId: string;
  userId: string;
}) {
  const interactions = await fetchUserInteractions(postId, userId);

  return (
    <div className="interactions">
      <button className={interactions.liked ? 'liked' : ''}>
        Like {interactions.likeCount}
      </button>
      <button>Comment {interactions.commentCount}</button>
      <button className={interactions.bookmarked ? 'bookmarked' : ''}>
        Bookmark
      </button>
    </div>
  );
}
```

## Optimizing PPR Performance

### Caching Strategy

```typescript
// lib/cache-config.ts
export const cacheConfig = {
  // Static content - long cache
  staticContent: {
    revalidate: 86400, // 24 hours
  },

  // Semi-static - medium cache
  semiStatic: {
    revalidate: 3600, // 1 hour
  },

  // User-specific - no cache
  userSpecific: {
    cache: 'no-store' as const,
  },

  // Shared dynamic - short cache
  sharedDynamic: {
    revalidate: 60, // 1 minute
  },
};

// Usage in components
async function ProductList() {
  const products = await fetch('https://api.example.com/products', {
    next: cacheConfig.semiStatic,
  }).then((r) => r.json());

  return <div>{/* Render products */}</div>;
}

async function UserData() {
  const user = await fetch('https://api.example.com/user', {
    ...cacheConfig.userSpecific,
  }).then((r) => r.json());

  return <div>{/* Render user */}</div>;
}
```

### Skeleton Component Patterns

```typescript
// components/skeletons.tsx
export function UserProfileSkeleton() {
  return (
    <div className="animate-pulse">
      <div className="flex items-center gap-4">
        <div className="w-16 h-16 bg-gray-200 rounded-full" />
        <div className="flex-1">
          <div className="h-4 bg-gray-200 rounded w-3/4 mb-2" />
          <div className="h-3 bg-gray-200 rounded w-1/2" />
        </div>
      </div>
    </div>
  );
}

export function StatCardSkeleton() {
  return (
    <div className="stat-card animate-pulse">
      <div className="h-4 bg-gray-200 rounded w-1/3 mb-4" />
      <div className="h-8 bg-gray-200 rounded w-1/2 mb-2" />
      <div className="h-3 bg-gray-200 rounded w-1/4" />
    </div>
  );
}

export function ContentSkeleton() {
  return (
    <div className="animate-pulse space-y-4">
      <div className="h-6 bg-gray-200 rounded w-2/3" />
      <div className="h-4 bg-gray-200 rounded" />
      <div className="h-4 bg-gray-200 rounded" />
      <div className="h-4 bg-gray-200 rounded w-5/6" />
    </div>
  );
}
```

## Monitoring PPR Performance

### Performance Tracking

```typescript
// lib/ppr-monitor.ts
interface PPRMetrics {
  pageId: string;
  staticShellTime: number;
  dynamicContentTime: number;
  totalTime: number;
  dynamicComponents: number;
}

class PPRMonitor {
  private metrics: Map<string, PPRMetrics> = new Map();

  startPage(pageId: string): void {
    this.metrics.set(pageId, {
      pageId,
      staticShellTime: performance.now(),
      dynamicContentTime: 0,
      totalTime: 0,
      dynamicComponents: 0,
    });
  }

  markShellComplete(pageId: string): void {
    const metric = this.metrics.get(pageId);
    if (metric) {
      metric.staticShellTime = performance.now() - metric.staticShellTime;
    }
  }

  markDynamicStart(pageId: string): void {
    const metric = this.metrics.get(pageId);
    if (metric) {
      metric.dynamicContentTime = performance.now();
    }
  }

  markDynamicComplete(pageId: string): void {
    const metric = this.metrics.get(pageId);
    if (metric) {
      metric.dynamicContentTime =
        performance.now() - metric.dynamicContentTime;
      metric.totalTime = metric.staticShellTime + metric.dynamicContentTime;
      metric.dynamicComponents++;
    }
  }

  getReport(pageId: string): PPRMetrics | undefined {
    return this.metrics.get(pageId);
  }

  sendToAnalytics(pageId: string): void {
    const metric = this.metrics.get(pageId);
    if (metric && typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', 'ppr_performance', {
        page_id: pageId,
        static_shell_time: metric.staticShellTime,
        dynamic_content_time: metric.dynamicContentTime,
        total_time: metric.totalTime,
        dynamic_components: metric.dynamicComponents,
      });
    }
  }
}

export const pprMonitor = new PPRMonitor();
```

## Common Mistakes

### 1. Too Many Dynamic Holes

```typescript
// BAD: Everything is dynamic (defeats the purpose)
<Suspense fallback={<Skeleton />}>
  <Header />
</Suspense>
<Suspense fallback={<Skeleton />}>
  <Navigation />
</Suspense>
<Suspense fallback={<Skeleton />}>
  <Content />
</Suspense>
<Suspense fallback={<Skeleton />}>
  <Footer />
</Suspense>

// GOOD: Only truly dynamic parts
<Header />
<Navigation />
<Content>
  <Suspense fallback={<Skeleton />}>
    <UserWidget /> {/* Only this is dynamic */}
  </Suspense>
</Content>
<Footer />
```

### 2. Not Caching Static Parts

```typescript
// BAD: Fetching static data without caching
async function ProductInfo() {
  const product = await fetch('https://api.example.com/product', {
    cache: 'no-store', // Don't do this for static data!
  }).then((r) => r.json());

  return <div>{product.name}</div>;
}

// GOOD: Cache static content
async function ProductInfo() {
  const product = await fetch('https://api.example.com/product', {
    next: { revalidate: 3600 }, // Cache for 1 hour
  }).then((r) => r.json());

  return <div>{product.name}</div>;
}
```

### 3. Poor Skeleton Design

```typescript
// BAD: Generic loading state
<Suspense fallback={<div>Loading...</div>}>
  <ComplexComponent />
</Suspense>

// GOOD: Layout-matching skeleton
<Suspense fallback={<ComplexComponentSkeleton />}>
  <ComplexComponent />
</Suspense>
```

## Best Practices

### 1. Identify Static vs Dynamic Content

```typescript
// Clearly separate concerns
function Page() {
  return (
    <>
      {/* Static: Can be pre-rendered */}
      <StaticHero />
      <StaticProductGrid />

      {/* Dynamic: User-specific */}
      <Suspense fallback={<UserSkeleton />}>
        <UserWelcome />
      </Suspense>

      {/* Static: Shared content */}
      <StaticNewsletter />

      {/* Dynamic: Personalized */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <PersonalizedRecommendations />
      </Suspense>
    </>
  );
}
```

### 2. Optimize Cache Strategy

```typescript
// Different cache strategies for different content types
const CACHE_STRATEGIES = {
  static: { revalidate: 86400 }, // 24 hours
  semiStatic: { revalidate: 3600 }, // 1 hour
  dynamic: { cache: 'no-store' as const },
};
```

### 3. Design Meaningful Skeletons

```typescript
// Match the layout of actual content
function ProductCardSkeleton() {
  return (
    <div className="product-card">
      <div className="h-48 bg-gray-200 rounded" /> {/* Image */}
      <div className="p-4">
        <div className="h-4 bg-gray-200 rounded mb-2" /> {/* Title */}
        <div className="h-3 bg-gray-200 rounded w-2/3" /> {/* Price */}
      </div>
    </div>
  );
}
```

## When to Use PPR

### Ideal Use Cases

1. **E-commerce sites** - Static product info + dynamic pricing/cart
2. **News sites** - Static articles + personalized recommendations
3. **Dashboards** - Static layout + dynamic user data
4. **Social platforms** - Static feed structure + dynamic interactions
5. **Marketing sites** - Static content + personalized CTAs

### Not Recommended For

1. **Fully static sites** - Use SSG instead
2. **Fully dynamic apps** - Use SSR or CSR
3. **Real-time applications** - Everything changes constantly
4. **Simple pages** - Overhead not justified

## Interview Questions

1. **What is Partial Pre-Rendering?**
   - Combines SSG and SSR in one page
   - Static shell pre-rendered at build time
   - Dynamic holes filled at request time
   - Best of both worlds

2. **How does PPR differ from ISR?**
   - ISR: entire page regenerated periodically
   - PPR: static shell + dynamic holes every request
   - PPR: better for personalized content

3. **What are the benefits of PPR?**
   - Fast TTFB (static shell from CDN)
   - Fresh dynamic content
   - Personalization support
   - Reduced server load

4. **How do you decide what should be static vs dynamic?**
   - Static: Shared content, rarely changes
   - Dynamic: User-specific, real-time data
   - Consider cache strategy

5. **What are the limitations of PPR?**
   - Requires React 18+ and Suspense
   - Next.js 14+ specific (currently)
   - Learning curve
   - Need to design good skeletons

## Key Takeaways

1. PPR combines benefits of SSG and SSR
2. Static shell delivered instantly from CDN
3. Dynamic holes filled at request time
4. Use Suspense to mark dynamic boundaries
5. Design skeletons that match actual content
6. Cache static parts appropriately
7. Only make truly dynamic parts dynamic
8. Great for e-commerce and personalized sites
9. Requires Next.js 14+ with experimental flag
10. Balance static and dynamic for optimal performance

## Resources

- [Next.js PPR Documentation](https://nextjs.org/docs/app/building-your-application/rendering/partial-prerendering)
- [PPR Announcement](https://nextjs.org/blog/next-14)
- [React Suspense](https://react.dev/reference/react/Suspense)
- [Vercel PPR Guide](https://vercel.com/blog/partial-prerendering-with-next-js-creating-a-new-default-rendering-model)
- [PPR Performance](https://web.dev/rendering-on-the-web/)
