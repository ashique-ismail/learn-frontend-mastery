# Streaming SSR (Server-Side Rendering)

## Overview

Streaming SSR is an advanced rendering technique that sends HTML to the client progressively, in chunks, rather than waiting for the entire page to render. This approach significantly improves Time to First Byte (TTFB) and allows users to see content faster, even while the rest of the page is still being generated.

## How Streaming SSR Works

```
Traditional SSR:
┌──────────────────────────────────────────────────────────────┐
│  Server Processing (Blocking)                                │
├──────────────────────────────────────────────────────────────┤
│  1. Fetch all data          (2s)                             │
│  2. Render entire page      (1s)                             │
│  3. Send complete HTML      (0.5s)                           │
│  Total: 3.5s before user sees anything                       │
└──────────────────────────────────────────────────────────────┘

Streaming SSR:
┌──────────────────────────────────────────────────────────────┐
│  Progressive Streaming                                        │
├──────────────────────────────────────────────────────────────┤
│  1. Send shell immediately  (0.1s) ← User sees layout!       │
│  2. Stream ready content    (0.5s) ← Main content appears    │
│  3. Stream slow data        (2s)   ← Final parts load        │
│  User sees content in 0.1s instead of 3.5s!                  │
└──────────────────────────────────────────────────────────────┘

Stream Flow:
┌─────────┐         ┌─────────────────────────────────────┐
│ Client  │         │ Server                              │
└────┬────┘         └─────┬───────────────────────────────┘
     │                     │
     │ Request             │
     ├────────────────────▶│ 1. Send Shell
     │                     │    <html><body><header>...
     │◀────────────────────┤
     │                     │
     │                     │ 2. Fetch & Stream Fast Data
     │◀────────────────────┤    <section>Content...</section>
     │                     │
     │                     │ 3. Fetch & Stream Slow Data
     │◀────────────────────┤    <aside>Sidebar...</aside>
     │                     │
     │                     │ 4. Close Stream
     │◀────────────────────┤    </body></html>
     │                     │
```

## React 18 Streaming with Suspense

### Basic Streaming Setup

```typescript
// app/layout.tsx (Next.js 13+ App Router)
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <header>
          <nav>Navigation (Sent immediately)</nav>
        </header>
        {children}
        <footer>Footer (Sent immediately)</footer>
      </body>
    </html>
  );
}
```

### Streaming with Suspense Boundaries

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { UserProfile } from './UserProfile';
import { RecentActivity } from './RecentActivity';
import { Analytics } from './Analytics';

export default function DashboardPage() {
  return (
    <div className="dashboard">
      {/* Header renders immediately */}
      <h1>Dashboard</h1>

      {/* User profile - fast query, renders quickly */}
      <Suspense fallback={<UserProfileSkeleton />}>
        <UserProfile />
      </Suspense>

      <div className="grid">
        {/* Recent activity - medium speed */}
        <Suspense fallback={<ActivitySkeleton />}>
          <RecentActivity />
        </Suspense>

        {/* Analytics - slow query, doesn't block page */}
        <Suspense fallback={<AnalyticsSkeleton />}>
          <Analytics />
        </Suspense>
      </div>
    </div>
  );
}

// Skeleton components for loading states
function UserProfileSkeleton() {
  return (
    <div className="skeleton animate-pulse">
      <div className="h-16 w-16 bg-gray-200 rounded-full" />
      <div className="h-4 w-32 bg-gray-200 rounded" />
    </div>
  );
}

function ActivitySkeleton() {
  return (
    <div className="skeleton animate-pulse">
      {[...Array(5)].map((_, i) => (
        <div key={i} className="h-12 bg-gray-200 rounded mb-2" />
      ))}
    </div>
  );
}

function AnalyticsSkeleton() {
  return (
    <div className="skeleton animate-pulse">
      <div className="h-64 bg-gray-200 rounded" />
    </div>
  );
}
```

### Async Server Components

```typescript
// app/dashboard/UserProfile.tsx
interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
}

// This is an async Server Component
export async function UserProfile() {
  // This fetch happens on the server
  const user = await fetchUser();

  return (
    <div className="user-profile">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

async function fetchUser(): Promise<User> {
  const response = await fetch('https://api.example.com/user', {
    // Next.js specific: cache for 60 seconds
    next: { revalidate: 60 },
  });

  if (!response.ok) {
    throw new Error('Failed to fetch user');
  }

  return response.json();
}
```

### Nested Suspense Boundaries

```typescript
// app/blog/[slug]/page.tsx
import { Suspense } from 'react';

interface PageProps {
  params: { slug: string };
}

export default function BlogPost({ params }: PageProps) {
  return (
    <article>
      {/* Post content - priority data */}
      <Suspense fallback={<PostSkeleton />}>
        <PostContent slug={params.slug} />
      </Suspense>

      {/* Sidebar with nested suspense */}
      <aside>
        {/* Author info - medium priority */}
        <Suspense fallback={<AuthorSkeleton />}>
          <AuthorInfo slug={params.slug} />
        </Suspense>

        {/* Related posts - lower priority */}
        <Suspense fallback={<RelatedSkeleton />}>
          <RelatedPosts slug={params.slug} />
        </Suspense>

        {/* Comments - lowest priority */}
        <Suspense fallback={<CommentsSkeleton />}>
          <Comments slug={params.slug} />
        </Suspense>
      </aside>
    </article>
  );
}

async function PostContent({ slug }: { slug: string }) {
  const post = await fetchPost(slug);
  return (
    <div>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </div>
  );
}

async function AuthorInfo({ slug }: { slug: string }) {
  const post = await fetchPost(slug);
  const author = await fetchAuthor(post.authorId);
  
  return (
    <div className="author-card">
      <img src={author.avatar} alt={author.name} />
      <h3>{author.name}</h3>
      <p>{author.bio}</p>
    </div>
  );
}

async function RelatedPosts({ slug }: { slug: string }) {
  const related = await fetchRelatedPosts(slug);
  
  return (
    <div className="related-posts">
      <h3>Related Posts</h3>
      {related.map(post => (
        <a key={post.slug} href={`/blog/${post.slug}`}>
          {post.title}
        </a>
      ))}
    </div>
  );
}

async function Comments({ slug }: { slug: string }) {
  const comments = await fetchComments(slug);
  
  return (
    <div className="comments">
      <h3>Comments ({comments.length})</h3>
      {comments.map(comment => (
        <div key={comment.id} className="comment">
          <strong>{comment.author}</strong>
          <p>{comment.text}</p>
        </div>
      ))}
    </div>
  );
}
```

## Custom Streaming Implementation

### Node.js Streaming SSR

```typescript
// server/stream-ssr.ts
import { Readable } from 'stream';
import { renderToNodeStream } from 'react-dom/server';
import React from 'react';
import express from 'express';

const app = express();

interface StreamChunk {
  id: string;
  component: React.ReactElement;
  priority: number;
}

class StreamingRenderer {
  private chunks: StreamChunk[] = [];

  addChunk(chunk: StreamChunk): void {
    this.chunks.push(chunk);
    // Sort by priority
    this.chunks.sort((a, b) => a.priority - b.priority);
  }

  async renderToStream(res: express.Response): Promise<void> {
    // Send initial shell
    res.write('<!DOCTYPE html><html><head>');
    res.write('<meta charset="utf-8">');
    res.write('<title>Streaming SSR</title>');
    res.write('</head><body>');
    res.write('<div id="root">');

    // Stream chunks in priority order
    for (const chunk of this.chunks) {
      await this.streamChunk(res, chunk);
    }

    res.write('</div>');
    res.write('<script src="/hydrate.js"></script>');
    res.write('</body></html>');
    res.end();
  }

  private async streamChunk(
    res: express.Response,
    chunk: StreamChunk
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const stream = renderToNodeStream(chunk.component);

      res.write(`<div id="${chunk.id}">`);

      stream.on('data', (data) => {
        res.write(data);
      });

      stream.on('end', () => {
        res.write('</div>');
        
        // Send script to hydrate this chunk
        res.write(`
          <script>
            if (window.hydrateChunk) {
              window.hydrateChunk('${chunk.id}');
            }
          </script>
        `);
        
        resolve();
      });

      stream.on('error', reject);
    });
  }
}

app.get('*', async (req, res) => {
  const renderer = new StreamingRenderer();

  // Add chunks with priorities
  renderer.addChunk({
    id: 'header',
    component: React.createElement(Header),
    priority: 1,
  });

  renderer.addChunk({
    id: 'main-content',
    component: React.createElement(MainContent),
    priority: 2,
  });

  renderer.addChunk({
    id: 'sidebar',
    component: React.createElement(Sidebar),
    priority: 3,
  });

  res.setHeader('Content-Type', 'text/html');
  res.setHeader('Transfer-Encoding', 'chunked');

  await renderer.renderToStream(res);
});

const port = 3000;
app.listen(port, () => {
  console.log(`Streaming SSR server running on port ${port}`);
});
```

### Progressive Hydration

```typescript
// client/progressive-hydration.ts
interface HydrationChunk {
  id: string;
  priority: number;
  hydrated: boolean;
}

class ProgressiveHydrator {
  private chunks: Map<string, HydrationChunk> = new Map();
  private observer: IntersectionObserver;

  constructor() {
    // Observe when chunks become visible
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            const chunkId = entry.target.id;
            this.hydrateChunk(chunkId);
          }
        });
      },
      { rootMargin: '50px' }
    );
  }

  registerChunk(id: string, priority: number): void {
    this.chunks.set(id, { id, priority, hydrated: false });

    const element = document.getElementById(id);
    if (element) {
      // High priority: hydrate immediately
      if (priority === 1) {
        this.hydrateChunk(id);
      } else {
        // Lower priority: hydrate when visible
        this.observer.observe(element);
      }
    }
  }

  async hydrateChunk(id: string): Promise<void> {
    const chunk = this.chunks.get(id);
    if (!chunk || chunk.hydrated) return;

    console.log(`Hydrating chunk: ${id}`);

    try {
      // Dynamic import of component
      const module = await import(`./chunks/${id}`);
      const Component = module.default;

      // Hydrate the chunk
      const container = document.getElementById(id);
      if (container) {
        const { hydrateRoot } = await import('react-dom/client');
        hydrateRoot(container, React.createElement(Component));
        
        chunk.hydrated = true;
        this.observer.unobserve(container);
      }
    } catch (error) {
      console.error(`Failed to hydrate chunk ${id}:`, error);
    }
  }

  hydrateAll(): void {
    // Hydrate all remaining chunks
    const sortedChunks = Array.from(this.chunks.values())
      .filter(c => !c.hydrated)
      .sort((a, b) => a.priority - b.priority);

    sortedChunks.forEach(chunk => {
      this.hydrateChunk(chunk.id);
    });
  }
}

// Global hydrator instance
export const hydrator = new ProgressiveHydrator();

// Expose to window for inline scripts
(window as any).hydrateChunk = (id: string) => {
  hydrator.hydrateChunk(id);
};
```

## Angular Universal Streaming (Experimental)

### Angular Streaming Configuration

```typescript
// server.ts
import 'zone.js/node';
import { ngExpressEngine } from '@nguniversal/express-engine';
import { ApplicationRef } from '@angular/core';
import { first } from 'rxjs/operators';
import express from 'express';

const app = express();

// Custom streaming engine
function streamingEngine(
  filePath: string,
  options: any,
  callback: (err: any, html?: string) => void
) {
  const { req, res } = options;

  // Send initial shell
  res.write('<!DOCTYPE html><html><head>');
  res.write('<meta charset="utf-8">');
  res.write('<title>Angular Streaming</title>');
  res.write('</head><body>');
  res.write('<app-root>');

  // Render Angular app
  ngExpressEngine({
    bootstrap: AppServerModule,
    providers: [
      {
        provide: 'REQUEST',
        useValue: req,
      },
      {
        provide: 'RESPONSE',
        useValue: res,
      },
    ],
  })(filePath, options, (err, html) => {
    if (err) {
      callback(err);
      return;
    }

    // Stream rendered content
    res.write(html);
    res.write('</app-root>');
    res.write('</body></html>');
    res.end();

    callback(null);
  });
}

app.engine('html', streamingEngine);
app.set('view engine', 'html');

app.get('*', (req, res) => {
  res.render('index', { req, res });
});
```

### Angular Component with Deferred Loading

```typescript
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <header>
      <h1>My App</h1>
      <nav>Navigation</nav>
    </header>

    <main>
      <!-- Immediately rendered -->
      <section class="hero">
        <h2>Welcome</h2>
      </section>

      <!-- Deferred loading -->
      <section class="content">
        <ng-container *ngIf="isServer">
          <div class="skeleton">Loading...</div>
        </ng-container>
        <ng-container *ngIf="!isServer">
          <app-dynamic-content></app-dynamic-content>
        </ng-container>
      </section>
    </main>

    <footer>Footer</footer>
  `
})
export class AppComponent {
  isServer = typeof window === 'undefined';

  ngOnInit() {
    // Load dynamic content client-side
    if (!this.isServer) {
      this.loadDynamicContent();
    }
  }

  private loadDynamicContent(): void {
    // Dynamic import
    import('./dynamic-content/dynamic-content.module').then(
      ({ DynamicContentModule }) => {
        // Module loaded and ready
        console.log('Dynamic content loaded');
      }
    );
  }
}
```

## Streaming with Data Fetching

### Parallel Data Streaming

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      
      {/* All these load in parallel and stream independently */}
      <div className="grid">
        <Suspense fallback={<Skeleton />}>
          <UserStats />
        </Suspense>

        <Suspense fallback={<Skeleton />}>
          <RecentOrders />
        </Suspense>

        <Suspense fallback={<Skeleton />}>
          <Revenue />
        </Suspense>

        <Suspense fallback={<Skeleton />}>
          <TopProducts />
        </Suspense>
      </div>
    </div>
  );
}

// Each component fetches independently
async function UserStats() {
  const stats = await fetch('https://api.example.com/stats/users').then(r => r.json());
  return <div>Users: {stats.total}</div>;
}

async function RecentOrders() {
  const orders = await fetch('https://api.example.com/orders/recent').then(r => r.json());
  return <div>Orders: {orders.length}</div>;
}

async function Revenue() {
  const revenue = await fetch('https://api.example.com/stats/revenue').then(r => r.json());
  return <div>Revenue: ${revenue.total}</div>;
}

async function TopProducts() {
  const products = await fetch('https://api.example.com/products/top').then(r => r.json());
  return (
    <ul>
      {products.map((p: any) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

### Waterfall Prevention

```typescript
// BAD: Creates waterfall (slow!)
async function BlogPost({ slug }: { slug: string }) {
  const post = await fetchPost(slug); // Wait for post
  const author = await fetchAuthor(post.authorId); // Then wait for author
  const comments = await fetchComments(post.id); // Then wait for comments
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>By {author.name}</p>
      <Comments data={comments} />
    </article>
  );
}

// GOOD: Parallel fetching with separate Suspense boundaries
function BlogPost({ slug }: { slug: string }) {
  return (
    <article>
      <Suspense fallback={<PostSkeleton />}>
        <PostContent slug={slug} />
      </Suspense>

      <Suspense fallback={<AuthorSkeleton />}>
        <AuthorInfo slug={slug} />
      </Suspense>

      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection slug={slug} />
      </Suspense>
    </article>
  );
}

// Each component fetches independently (parallel!)
async function PostContent({ slug }: { slug: string }) {
  const post = await fetchPost(slug);
  return <div><h1>{post.title}</h1><p>{post.content}</p></div>;
}

async function AuthorInfo({ slug }: { slug: string }) {
  // Fetches post and author in parallel with PostContent
  const post = await fetchPost(slug);
  const author = await fetchAuthor(post.authorId);
  return <p>By {author.name}</p>;
}

async function CommentsSection({ slug }: { slug: string }) {
  // Fetches comments in parallel with other components
  const post = await fetchPost(slug);
  const comments = await fetchComments(post.id);
  return <Comments data={comments} />;
}
```

## Error Handling in Streaming

### Error Boundaries with Suspense

```typescript
// app/error.tsx (Next.js Error Boundary)
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div className="error-boundary">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// Usage with Suspense
function MyPage() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Graceful Fallbacks

```typescript
// components/SafeComponent.tsx
import { Suspense } from 'react';

interface SafeComponentProps {
  children: React.ReactNode;
  fallback: React.ReactNode;
  errorFallback: React.ReactNode;
}

export function SafeComponent({
  children,
  fallback,
  errorFallback,
}: SafeComponentProps) {
  return (
    <ErrorBoundary fallback={errorFallback}>
      <Suspense fallback={fallback}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

// Usage
function Dashboard() {
  return (
    <SafeComponent
      fallback={<Skeleton />}
      errorFallback={<ErrorMessage />}
    >
      <DataComponent />
    </SafeComponent>
  );
}
```

## Performance Optimization

### Streaming Metrics

```typescript
// lib/streaming-metrics.ts
interface StreamMetric {
  chunkId: string;
  startTime: number;
  endTime?: number;
  duration?: number;
  size?: number;
}

class StreamingMetrics {
  private metrics: Map<string, StreamMetric> = new Map();

  startChunk(chunkId: string): void {
    this.metrics.set(chunkId, {
      chunkId,
      startTime: performance.now(),
    });
  }

  endChunk(chunkId: string, size?: number): void {
    const metric = this.metrics.get(chunkId);
    if (metric) {
      metric.endTime = performance.now();
      metric.duration = metric.endTime - metric.startTime;
      metric.size = size;
    }
  }

  getMetrics(): StreamMetric[] {
    return Array.from(this.metrics.values());
  }

  getSummary() {
    const metrics = this.getMetrics();
    const totalDuration = metrics.reduce((sum, m) => sum + (m.duration || 0), 0);
    const avgDuration = totalDuration / metrics.length;

    return {
      totalChunks: metrics.length,
      totalDuration,
      avgDuration,
      chunks: metrics,
    };
  }

  sendToAnalytics(): void {
    const summary = this.getSummary();
    
    // Send to analytics service
    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', 'streaming_metrics', {
        total_chunks: summary.totalChunks,
        total_duration: summary.totalDuration,
        avg_duration: summary.avgDuration,
      });
    }
  }
}

export const streamingMetrics = new StreamingMetrics();
```

## Common Mistakes

### 1. Too Many Suspense Boundaries

```typescript
// BAD: Over-granular boundaries
function Page() {
  return (
    <div>
      <Suspense fallback={<div>Loading title...</div>}>
        <Title />
      </Suspense>
      <Suspense fallback={<div>Loading subtitle...</div>}>
        <Subtitle />
      </Suspense>
      <Suspense fallback={<div>Loading content...</div>}>
        <Content />
      </Suspense>
      {/* Too many boundaries! */}
    </div>
  );
}

// GOOD: Logical boundaries
function Page() {
  return (
    <div>
      {/* Group related content */}
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}
```

### 2. Blocking Data Fetches

```typescript
// BAD: Sequential fetching
async function Page() {
  const user = await fetchUser();
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);
  // This creates a waterfall!
  
  return <div>...</div>;
}

// GOOD: Use separate boundaries
function Page() {
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <User />
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
    </div>
  );
}
```

### 3. Missing Loading States

```typescript
// BAD: No fallback
<Suspense>
  <SlowComponent />
</Suspense>

// GOOD: Meaningful fallback
<Suspense fallback={<ComponentSkeleton />}>
  <SlowComponent />
</Suspense>
```

## Best Practices

### 1. Strategic Suspense Placement

```typescript
// Place boundaries at logical component boundaries
function Dashboard() {
  return (
    <div>
      {/* Critical content */}
      <header>Dashboard</header>

      {/* Independent sections with own boundaries */}
      <Suspense fallback={<StatsLoading />}>
        <Statistics />
      </Suspense>

      <Suspense fallback={<ChartLoading />}>
        <Charts />
      </Suspense>

      <Suspense fallback={<TableLoading />}>
        <DataTable />
      </Suspense>
    </div>
  );
}
```

### 2. Prioritize Critical Content

```typescript
// Send critical content first, defer less important
function ProductPage() {
  return (
    <>
      {/* Above-the-fold content: no suspense */}
      <ProductImages />
      <ProductTitle />
      <Price />
      <AddToCartButton />

      {/* Below-the-fold: can stream later */}
      <Suspense fallback={<DescriptionSkeleton />}>
        <ProductDescription />
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews />
      </Suspense>

      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts />
      </Suspense>
    </>
  );
}
```

### 3. Implement Meaningful Skeletons

```typescript
// Match skeleton to actual content layout
function ProductSkeleton() {
  return (
    <div className="animate-pulse">
      <div className="h-64 bg-gray-200 rounded" />
      <div className="h-8 bg-gray-200 rounded mt-4 w-3/4" />
      <div className="h-4 bg-gray-200 rounded mt-2 w-1/2" />
      <div className="h-12 bg-gray-200 rounded mt-4" />
    </div>
  );
}
```

## When to Use Streaming SSR

### Ideal Use Cases

1. **Content-heavy pages** - Long articles, documentation
2. **Dashboard applications** - Multiple independent data sources
3. **E-commerce** - Product listings with various data
4. **Social feeds** - Posts loading incrementally
5. **Data visualization** - Charts loading independently

### Not Recommended For

1. **Simple pages** - Overhead not worth it
2. **Highly interactive apps** - Better with CSR
3. **Real-time collaboration** - Use WebSocket instead
4. **Authentication flows** - Need complete data upfront

## Interview Questions

1. **What is streaming SSR?**
   - Progressive HTML rendering and sending
   - Chunks sent as they're ready
   - Improves TTFB and perceived performance

2. **How does Suspense enable streaming?**
   - Marks boundaries where React can pause rendering
   - Sends fallback immediately
   - Streams actual content when ready

3. **What are the benefits of streaming SSR?**
   - Faster Time to First Byte
   - Better perceived performance
   - Progressive content loading
   - Doesn't block on slow data

4. **What is the difference between streaming SSR and traditional SSR?**
   - Traditional: wait for all data, send complete HTML
   - Streaming: send shell immediately, stream content progressively

5. **How do you handle errors in streaming SSR?**
   - Error Boundaries around Suspense
   - Graceful fallbacks
   - Error state components

## Key Takeaways

1. Streaming SSR sends HTML progressively in chunks
2. React Suspense enables streaming boundaries
3. Users see content faster without waiting for full page
4. Place Suspense boundaries strategically
5. Parallel data fetching prevents waterfalls
6. Use meaningful loading skeletons
7. Prioritize above-the-fold content
8. Implement error boundaries
9. Monitor streaming metrics
10. Not suitable for all applications

## Resources

- [React 18 Suspense](https://react.dev/reference/react/Suspense)
- [Next.js Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [Streaming SSR Guide](https://www.patterns.dev/posts/ssr/)
- [React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023)
- [Web.dev Streaming](https://web.dev/streaming-ssr/)
