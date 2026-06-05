# Streaming SSR with Suspense

## The Idea

**In plain English:** Streaming SSR is a way for a web server to send a webpage to your browser piece by piece as each section becomes ready, instead of making you wait until the entire page is fully built. "SSR" stands for Server-Side Rendering, which means the server builds the initial HTML before sending it to you.

**Real-world analogy:** Imagine ordering food at a restaurant where the kitchen sends each dish to your table the moment it is ready, rather than holding everything in the kitchen until every single item is plated. A waiter brings your bread and drinks immediately, then your starter when it is done, then your main course, and finally dessert — you never sit there staring at an empty table.

- The empty table with placeholders (bread basket, empty glasses) = the skeleton/fallback UI shown while data loads
- Each dish arriving from the kitchen as it finishes = a component streaming to the browser when its data is ready
- The waiter delivering dishes in the order they are cooked (not the order you ordered) = out-of-order streaming based on which fetch completes first

---

## Table of Contents

- [Introduction](#introduction)
- [Traditional SSR vs Streaming SSR](#traditional-ssr-vs-streaming-ssr)
- [How Streaming Works](#how-streaming-works)
- [Suspense for Data Fetching](#suspense-for-data-fetching)
- [Selective Hydration](#selective-hydration)
- [Streaming APIs](#streaming-apis)
- [Out-of-Order Streaming](#out-of-order-streaming)
- [Streaming Boundaries](#streaming-boundaries)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Streaming SSR allows the server to send HTML to the client progressively, chunk by chunk, rather than waiting for the entire page to render. Combined with Suspense, it enables selective hydration, faster TTFB (Time to First Byte), and improved perceived performance.

## Traditional SSR vs Streaming SSR

### Traditional SSR Waterfall

```text
Server:
1. Fetch all data ────────────────── (4s)
2. Render entire HTML ───────────── (2s)
3. Send HTML ─ (1s)

Client:
4. Load JavaScript ───────────────── (3s)
5. Hydrate entire app ─────────────── (2s)

Total: 12 seconds until interactive
User sees nothing for 7 seconds
```

**Traditional SSR Code:**

```typescript
// pages/index.tsx (Next.js Pages Router)
export async function getServerSideProps() {
  // Must wait for ALL data before rendering
  const [user, posts, comments, ads] = await Promise.all([
    fetchUser(),       // 1s
    fetchPosts(),      // 3s
    fetchComments(),   // 2s
    fetchAds()         // 1s
  ]);
  // Total wait: 3s (slowest request)

  return {
    props: { user, posts, comments, ads }
  };
}

export default function Page({ user, posts, comments, ads }) {
  // Entire page renders at once
  // User sees nothing until all data arrives
  return (
    <>
      <Header user={user} />
      <Posts posts={posts} />
      <Comments comments={comments} />
      <Sidebar ads={ads} />
    </>
  );
}
```

### Streaming SSR Flow

```text
Server:
1. Send shell HTML immediately ─ (100ms)
2. Stream Posts ───── (1s)
3. Stream Comments ─── (2s)
4. Stream Sidebar ──── (3s)

Client:
1. Show shell immediately ─ (100ms)
2. Hydrate Posts ───── (1s + 500ms)
3. Hydrate Comments ─── (2s + 500ms)
4. Hydrate Sidebar ──── (3s + 500ms)

User sees content in 100ms!
Progressive interactivity
```

**Streaming SSR Code:**

```typescript
// app/page.tsx (Next.js App Router)
export default function Page() {
  // Shell renders immediately
  return (
    <>
      <Header /> {/* Static, renders immediately */}
      
      {/* Posts stream when ready */}
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
      
      {/* Comments stream independently */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments />
      </Suspense>
      
      {/* Sidebar streams last */}
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
    </>
  );
}

// Each component fetches its own data
async function Posts() {
  const posts = await fetchPosts(); // Doesn't block others
  return <PostList posts={posts} />;
}

async function Comments() {
  const comments = await fetchComments(); // Independent
  return <CommentList comments={comments} />;
}

async function Sidebar() {
  const ads = await fetchAds(); // Streams last
  return <AdList ads={ads} />;
}
```

## How Streaming Works

### The Streaming Process

```typescript
/*
┌─────────────────────────────────────────────────────────┐
│                    SERVER STREAMING                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Initial HTML (shell) ─────────────────────► Client │
│     <!DOCTYPE html>                                     │
│     <html>                                              │
│       <body>                                            │
│         <div id="root">                                 │
│           <header>...</header>                          │
│           <div id="suspense-1">                         │
│             <!-- Fallback skeleton -->                  │
│           </div>                                        │
│                                                         │
│  2. Stream chunk when ready ──────────────────► Client │
│     <template id="suspense-1-data">                     │
│       <div>Actual content...</div>                      │
│     </template>                                         │
│     <script>                                            │
│       // Replace fallback with content                 │
│     </script>                                           │
│                                                         │
│  3. More chunks as they complete ─────────────► Client │
│                                                         │
└─────────────────────────────────────────────────────────┘
*/
```

### Streaming Example

```typescript
// app/dashboard/page.tsx
export default function Dashboard() {
  return (
    <div>
      {/* Renders immediately */}
      <h1>Dashboard</h1>
      <UserGreeting />

      <div className="grid">
        {/* Each Suspense boundary streams independently */}
        
        <Suspense fallback={<StatsSkeleton />}>
          <Stats />
        </Suspense>

        <Suspense fallback={<ChartSkeleton />}>
          <RevenueChart />
        </Suspense>

        <Suspense fallback={<TableSkeleton />}>
          <RecentOrders />
        </Suspense>

        <Suspense fallback={<ActivitySkeleton />}>
          <ActivityFeed />
        </Suspense>
      </div>
    </div>
  );
}

// These stream as they complete
async function Stats() {
  const stats = await db.stats.aggregate(); // 500ms
  return <StatsCards stats={stats} />;
}

async function RevenueChart() {
  const revenue = await db.orders.revenue(); // 1s
  return <Chart data={revenue} />;
}

async function RecentOrders() {
  const orders = await db.orders.recent(); // 2s
  return <OrderTable orders={orders} />;
}

async function ActivityFeed() {
  const activity = await db.activity.recent(); // 3s
  return <ActivityList items={activity} />;
}
```

## Suspense for Data Fetching

### Basic Suspense Pattern

```typescript
import { Suspense } from 'react';

function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <DataComponent />
    </Suspense>
  );
}

// Async Server Component
async function DataComponent() {
  const data = await fetchData();
  return <Display data={data} />;
}

function Loading() {
  return <div>Loading...</div>;
}
```

### Nested Suspense Boundaries

```typescript
function BlogPost({ id }: { id: string }) {
  return (
    <article>
      {/* Post content loads first */}
      <Suspense fallback={<PostSkeleton />}>
        <PostContent id={id} />
      </Suspense>

      {/* Comments load independently */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={id} />
        
        {/* Nested: Likes load after comments */}
        <Suspense fallback={<LikesSkeleton />}>
          <CommentLikes postId={id} />
        </Suspense>
      </Suspense>

      {/* Related posts load last */}
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedPosts postId={id} />
      </Suspense>
    </article>
  );
}

async function PostContent({ id }: { id: string }) {
  const post = await db.post.findUnique({ where: { id } });
  return (
    <>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </>
  );
}

async function Comments({ postId }: { postId: string }) {
  const comments = await db.comment.findMany({ where: { postId } });
  return <CommentList comments={comments} />;
}

async function CommentLikes({ postId }: { postId: string }) {
  const likes = await db.like.count({ where: { postId } });
  return <div>{likes} likes</div>;
}

async function RelatedPosts({ postId }: { postId: string }) {
  const related = await db.post.findRelated(postId);
  return <PostGrid posts={related} />;
}
```

### Parallel Data Fetching with Suspense

```typescript
function ProductPage({ id }: { id: string }) {
  return (
    <>
      {/* All three start fetching in parallel */}
      <Suspense fallback={<ProductSkeleton />}>
        <ProductInfo id={id} />
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews id={id} />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations id={id} />
      </Suspense>
    </>
  );
}

// All three fetch in parallel
async function ProductInfo({ id }: { id: string }) {
  const product = await fetchProduct(id); // 1s
  return <ProductDetail product={product} />;
}

async function ProductReviews({ id }: { id: string }) {
  const reviews = await fetchReviews(id); // 2s
  return <ReviewList reviews={reviews} />;
}

async function Recommendations({ id }: { id: string }) {
  const recs = await fetchRecommendations(id); // 3s
  return <ProductGrid products={recs} />;
}

// Timeline:
// 0s: All three start fetching
// 1s: ProductInfo streams
// 2s: ProductReviews streams
// 3s: Recommendations streams
```

## Selective Hydration

React 18 can hydrate parts of the page before others, prioritizing user interactions.

### How Selective Hydration Works

```typescript
function Page() {
  return (
    <html>
      <body>
        <Header /> {/* Hydrates first (above the fold) */}

        <Suspense fallback={<MainSkeleton />}>
          <MainContent /> {/* Hydrates when streamed */}
        </Suspense>

        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar /> {/* Hydrates when streamed */}
        </Suspense>

        <Suspense fallback={<CommentsSkeleton />}>
          <Comments /> {/* Hydrates when streamed or clicked */}
        </Suspense>
      </body>
    </html>
  );
}

/*
Hydration Order:
1. Header (immediate - no Suspense)
2. MainContent (when HTML streams)
3. Sidebar (when HTML streams)
4. Comments (when HTML streams OR user clicks)

If user clicks Comments before it's hydrated:
- React prioritizes Comments hydration
- Other boundaries wait
*/
```

### Prioritized Hydration on Interaction

```typescript
'use client';

function CommentSection() {
  const [expanded, setExpanded] = useState(false);

  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>
        Show Comments
      </button>

      {/* If user clicks before hydration completes,
          React prioritizes hydrating this component */}
      <Suspense fallback={<CommentsSkeleton />}>
        {expanded && <Comments />}
      </Suspense>
    </div>
  );
}

/*
Scenario:
1. Page loads, Comments HTML streams
2. JavaScript for Comments still loading
3. User clicks "Show Comments"
4. React prioritizes Comments hydration
5. Other components wait
6. User can interact with Comments immediately
*/
```

### Avoiding Hydration Waterfalls

```typescript
// ❌ BAD: Sequential hydration
function Page() {
  return (
    <Suspense fallback={<ShellSkeleton />}>
      <Shell>
        <Suspense fallback={<ContentSkeleton />}>
          <Content>
            <Suspense fallback={<DetailsSkeleton />}>
              <Details />
            </Suspense>
          </Content>
        </Suspense>
      </Shell>
    </Suspense>
  );
}
// Hydration: Shell → Content → Details (waterfall)

// ✅ GOOD: Parallel hydration
function Page() {
  return (
    <>
      <Shell /> {/* No Suspense, hydrates immediately */}

      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>

      <Suspense fallback={<DetailsSkeleton />}>
        <Details />
      </Suspense>
    </>
  );
}
// Hydration: Shell, Content, and Details in parallel
```

## Streaming APIs

### renderToPipeableStream (Node.js)

```typescript
// server.ts (Node.js)
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe, abort } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/main.js'],
    
    onShellReady() {
      // Shell is ready - start streaming
      res.statusCode = 200;
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    
    onShellError(error) {
      // Error in shell - send error page
      res.statusCode = 500;
      res.send('<h1>Server Error</h1>');
    },
    
    onAllReady() {
      // Everything rendered (for crawlers)
    },
    
    onError(error) {
      console.error('Streaming error:', error);
    }
  });

  // Abort after timeout
  setTimeout(() => abort(), 10000);
});
```

### renderToReadableStream (Web Streams)

```typescript
// Edge runtime (Cloudflare Workers, Deno, etc.)
import { renderToReadableStream } from 'react-dom/server';

export default async function handler(request: Request) {
  const stream = await renderToReadableStream(<App />, {
    bootstrapScripts: ['/main.js'],
    
    onError(error) {
      console.error('Stream error:', error);
    }
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/html',
      'Transfer-Encoding': 'chunked'
    }
  });
}
```

### Next.js App Router (Automatic Streaming)

```typescript
// app/page.tsx
// Streaming is automatic with Suspense
export default function Page() {
  return (
    <>
      <Header />
      
      <Suspense fallback={<ProductsSkeleton />}>
        <Products />
      </Suspense>
      
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews />
      </Suspense>
    </>
  );
}

// Next.js handles streaming automatically
// No need for renderToPipeableStream
```

## Out-of-Order Streaming

Components can stream in any order based on which data resolves first.

### Unordered Streaming Example

```typescript
function Dashboard() {
  return (
    <div className="dashboard">
      {/* These stream in whatever order they complete */}
      
      <Suspense fallback={<StatsCardSkeleton />}>
        <StatsCard /> {/* Completes in 3s */}
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart /> {/* Completes in 1s - streams first! */}
      </Suspense>

      <Suspense fallback={<TableSkeleton />}>
        <OrdersTable /> {/* Completes in 2s - streams second */}
      </Suspense>

      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed /> {/* Completes in 5s - streams last */}
      </Suspense>
    </div>
  );
}

async function StatsCard() {
  await sleep(3000);
  const stats = await db.stats.get();
  return <div>Stats: {stats.total}</div>;
}

async function RevenueChart() {
  await sleep(1000);
  const revenue = await db.revenue.get();
  return <Chart data={revenue} />;
}

async function OrdersTable() {
  await sleep(2000);
  const orders = await db.orders.recent();
  return <Table data={orders} />;
}

async function ActivityFeed() {
  await sleep(5000);
  const activity = await db.activity.get();
  return <Feed items={activity} />;
}

/*
Timeline:
0s:   Shell + all skeletons render
1s:   RevenueChart streams (completes first)
2s:   OrdersTable streams (completes second)
3s:   StatsCard streams (completes third)
5s:   ActivityFeed streams (completes last)

NOT in document order!
*/
```

### Controlling Stream Order

```typescript
// Force sequential order with nested Suspense
function SequentialDashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <RevenueChart />
      
      <Suspense fallback={<TableSkeleton />}>
        <OrdersTable />
        
        <Suspense fallback={<FeedSkeleton />}>
          <ActivityFeed />
        </Suspense>
      </Suspense>
    </Suspense>
  );
}
// Streams: Chart → Table → Feed (guaranteed order)

// Parallel groups
function GroupedDashboard() {
  return (
    <>
      {/* Group 1: Critical data (wait for both) */}
      <Suspense fallback={<CriticalSkeleton />}>
        <StatsCard />
        <RevenueChart />
      </Suspense>

      {/* Group 2: Secondary data (independent) */}
      <Suspense fallback={<TableSkeleton />}>
        <OrdersTable />
      </Suspense>

      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />
      </Suspense>
    </>
  );
}
// Group 1 streams when both StatsCard AND RevenueChart complete
// Group 2 and 3 stream independently
```

## Streaming Boundaries

### Choosing Boundary Granularity

```typescript
// ❌ TOO COARSE: One boundary for everything
function PageTooCoarse() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Header />
      <MainContent />
      <Sidebar />
      <Footer />
    </Suspense>
  );
}
// Problem: Everything waits for slowest component

// ❌ TOO FINE: Boundary for every tiny piece
function PageTooFine() {
  return (
    <>
      <Suspense fallback={<div>...</div>}>
        <Logo />
      </Suspense>
      <Suspense fallback={<div>...</div>}>
        <NavItem label="Home" />
      </Suspense>
      <Suspense fallback={<div>...</div>}>
        <NavItem label="About" />
      </Suspense>
      {/* 50 more Suspense boundaries... */}
    </>
  );
}
// Problem: Too many boundaries, overhead, poor UX

// ✅ JUST RIGHT: Logical boundaries
function PageJustRight() {
  return (
    <>
      <Header /> {/* Static, no Suspense */}

      <Suspense fallback={<MainContentSkeleton />}>
        <MainContent /> {/* One boundary for main area */}
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar /> {/* Separate boundary for sidebar */}
      </Suspense>

      <Footer /> {/* Static, no Suspense */}
    </>
  );
}
// Just right: Logical sections, good UX
```

### Boundary Patterns

```typescript
// Pattern 1: Above/below fold
function Page() {
  return (
    <>
      {/* Above fold: no Suspense, immediate */}
      <Hero />
      <CallToAction />

      {/* Below fold: can stream */}
      <Suspense fallback={<FeaturesSkeleton />}>
        <Features />
      </Suspense>

      <Suspense fallback={<TestimonialsSkeleton />}>
        <Testimonials />
      </Suspense>
    </>
  );
}

// Pattern 2: Critical vs non-critical
function Dashboard() {
  return (
    <>
      {/* Critical: no Suspense, must load immediately */}
      <NavigationBar />
      <UserProfile />

      {/* Non-critical: can stream */}
      <Suspense fallback={<AnalyticsSkeleton />}>
        <AnalyticsDashboard />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations />
      </Suspense>
    </>
  );
}

// Pattern 3: Nested by dependency
function ProductPage({ id }: { id: string }) {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductInfo id={id} />

      {/* Nested: Only loads after ProductInfo */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews productId={id} />
      </Suspense>
    </Suspense>
  );
}
```

## Common Mistakes

### 1. Wrapping Static Content in Suspense

```typescript
// ❌ BAD: Static content doesn't need Suspense
function Page() {
  return (
    <Suspense fallback={<HeaderSkeleton />}>
      <header>
        <h1>My Site</h1>
        <nav>
          <a href="/">Home</a>
          <a href="/about">About</a>
        </nav>
      </header>
    </Suspense>
  );
}

// ✅ GOOD: Only wrap async data
function Page() {
  return (
    <>
      <header>
        <h1>My Site</h1>
        <nav>
          <a href="/">Home</a>
          <a href="/about">About</a>
        </nav>
      </header>

      <Suspense fallback={<ContentSkeleton />}>
        <AsyncContent />
      </Suspense>
    </>
  );
}
```

### 2. Not Providing Fallbacks

```typescript
// ❌ BAD: No fallback (shows nothing)
function Page() {
  return (
    <Suspense>
      <AsyncContent />
    </Suspense>
  );
}

// ✅ GOOD: Meaningful fallback
function Page() {
  return (
    <Suspense fallback={<ContentSkeleton />}>
      <AsyncContent />
    </Suspense>
  );
}
```

### 3. Waterfall Suspense Boundaries

```typescript
// ❌ BAD: Sequential loading
function Page() {
  return (
    <Suspense fallback={<Skeleton1 />}>
      <Component1 />
      <Suspense fallback={<Skeleton2 />}>
        <Component2 />
        <Suspense fallback={<Skeleton3 />}>
          <Component3 />
        </Suspense>
      </Suspense>
    </Suspense>
  );
}

// ✅ GOOD: Parallel loading
function Page() {
  return (
    <>
      <Suspense fallback={<Skeleton1 />}>
        <Component1 />
      </Suspense>
      
      <Suspense fallback={<Skeleton2 />}>
        <Component2 />
      </Suspense>
      
      <Suspense fallback={<Skeleton3 />}>
        <Component3 />
      </Suspense>
    </>
  );
}
```

## Best Practices

### 1. Design Skeleton States

```typescript
// Good skeleton that matches final layout
function ProductCardSkeleton() {
  return (
    <div className="product-card">
      <div className="skeleton-image" />
      <div className="skeleton-title" />
      <div className="skeleton-price" />
      <div className="skeleton-button" />
    </div>
  );
}

function ProductCard({ id }: { id: string }) {
  return (
    <Suspense fallback={<ProductCardSkeleton />}>
      <ProductCardContent id={id} />
    </Suspense>
  );
}
```

### 2. Stream Non-Critical Content

```typescript
function Page() {
  return (
    <>
      {/* Critical: Render immediately */}
      <Header />
      <MainContent />

      {/* Non-critical: Stream later */}
      <Suspense fallback={null}>
        <AnalyticsTracking />
      </Suspense>

      <Suspense fallback={null}>
        <Recommendations />
      </Suspense>
    </>
  );
}
```

### 3. Use Loading Component for Transitions

```typescript
// app/posts/loading.tsx
export default function Loading() {
  return <PostListSkeleton />;
}

// app/posts/page.tsx
export default async function PostsPage() {
  const posts = await fetchPosts();
  return <PostList posts={posts} />;
}

// Next.js automatically wraps in Suspense with loading.tsx as fallback
```

## Interview Questions

### Q1: What is streaming SSR and how does it improve performance?

**Answer:** Streaming SSR sends HTML to the client progressively instead of waiting for the entire page to render. The server sends the shell immediately, then streams components as they complete. This improves TTFB (Time to First Byte), shows content faster, and enables selective hydration where React can prioritize hydrating interactive components.

### Q2: How does Suspense enable streaming?

**Answer:** Suspense creates streaming boundaries. When React encounters a Suspense boundary during server rendering, it immediately sends the fallback HTML, continues streaming other content, and later streams the actual component when its data is ready. Multiple Suspense boundaries can stream in parallel or out of order.

### Q3: What is selective hydration?

**Answer:** Selective hydration allows React to hydrate different parts of the page independently. With streaming SSR and Suspense, components hydrate as their HTML streams in. If a user interacts with a component before it's fully hydrated, React prioritizes hydrating that component first, making the app feel more responsive.

### Q4: How do you prevent hydration waterfalls?

**Answer:** Avoid nesting Suspense boundaries unnecessarily. Place Suspense boundaries as siblings rather than nested to allow parallel streaming and hydration. Only nest when there's a true dependency between components.

### Q5: What's the difference between renderToPipeableStream and renderToReadableStream?

**Answer:** `renderToPipeableStream` is for Node.js environments and returns a Node.js stream. `renderToReadableStream` is for Web Streams API (Edge runtime, Cloudflare Workers, Deno) and returns a ReadableStream. Both enable streaming SSR, just for different runtime environments.

## Key Takeaways

1. **Streaming sends HTML progressively** - Shell first, then components as they complete
2. **Suspense creates streaming boundaries** - Each boundary can stream independently
3. **Out-of-order streaming** - Components stream when ready, not in document order
4. **Selective hydration** - React hydrates components as they stream, prioritizing interactions
5. **Better perceived performance** - Users see content faster even if total time is the same
6. **Granular boundaries** - Too few is slow, too many is complex, find the right balance
7. **Design good skeletons** - Fallbacks should match final layout to avoid layout shift
8. **Works with concurrent features** - Streaming enables startTransition and other concurrent features

## Resources

### Official Documentation

- [Streaming SSR in React 18](https://github.com/reactwg/react-18/discussions/37)
- [Suspense for Data Fetching](https://react.dev/reference/react/Suspense)
- [Next.js Streaming and Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)

### Articles

- "New Suspense SSR Architecture in React 18" - React Team
- "Streaming Server Rendering with Suspense" - Dan Abramov
- "Selective Hydration in React 18" - Web.dev

### Videos

- "React 18 Keynote" - React Conf 2021
- "Streaming SSR Deep Dive" - Remix

---

Last Updated: 2026-05 - React 19 current
