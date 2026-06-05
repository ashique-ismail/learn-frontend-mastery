# Incremental Static Regeneration (ISR)

## The Idea

**In plain English:** ISR is a way to build web pages ahead of time (so they load fast), but automatically refresh them in the background after a set amount of time — so visitors always get a quick response while the site quietly updates itself with new content.

**Real-world analogy:** Think of a newspaper stand that stocks the morning edition at 6am. When you walk up, you get a copy instantly — no waiting for it to be printed. If you walk up at noon, the stand still gives you the morning edition immediately, but also quietly orders a fresh afternoon edition for the next person. By the time the next customer arrives, the updated paper is ready.

- The pre-printed morning edition = the pre-built static HTML page served from cache
- The newspaper stand handing you a copy instantly = the CDN serving the page with no delay
- The stand quietly ordering a fresh edition in the background = the server regenerating the page after the revalidation interval expires

---

## Overview

Incremental Static Regeneration (ISR) combines the benefits of static generation with the ability to update content without rebuilding the entire site. ISR allows you to create or update static pages after build time, on-demand or at specified intervals, providing a balance between performance and freshness.

## How ISR Works

```
Traditional SSG Flow:
┌─────────────┐    Build    ┌──────────┐    Deploy   ┌─────────┐
│   Content   │──────────▶│  Static  │──────────▶│   CDN   │
│   Changes   │            │   Pages  │            │  Serves │
└─────────────┘            └──────────┘            └─────────┘
    (Requires full rebuild to update)

ISR Flow:
┌─────────────────────────────────────────────────────────────┐
│  Initial Build: Generate static pages                       │
├─────────────────────────────────────────────────────────────┤
│  Runtime:                                                    │
│  1. User requests page                                       │
│  2. Serve stale content immediately (fast!)                 │
│  3. Check if revalidation needed (background)               │
│  4. If needed, regenerate page                              │
│  5. Replace old version with new                            │
│  6. Subsequent users get fresh content                      │
└─────────────────────────────────────────────────────────────┘

Request Flow:
┌────────┐      ┌─────────┐      ┌──────────────┐
│ User 1 │─────▶│ CDN     │─────▶│ Stale Page   │ (Instant)
└────────┘      └────┬────┘      └──────────────┘
                     │
                     │ (Background regeneration)
                     ▼
              ┌──────────────┐
              │ Fresh Page   │
              └──────┬───────┘
                     │
┌────────┐      ┌───▼─────┐
│ User 2 │─────▶│ CDN     │─────▶ Fresh Content!
└────────┘      └─────────┘
```

## Next.js ISR Implementation

### Basic Time-Based Revalidation

```typescript
// pages/blog/[slug].tsx
import { GetStaticPaths, GetStaticProps } from 'next';

interface Post {
  slug: string;
  title: string;
  content: string;
  updatedAt: string;
}

interface BlogPostProps {
  post: Post;
  revalidatedAt: number;
}

export const getStaticPaths: GetStaticPaths = async () => {
  // Pre-generate top 100 posts
  const topPosts = await fetch('https://api.example.com/posts?limit=100')
    .then(r => r.json());

  const paths = topPosts.map((post: Post) => ({
    params: { slug: post.slug },
  }));

  return {
    paths,
    fallback: 'blocking', // Generate other pages on-demand
  };
};

export const getStaticProps: GetStaticProps<BlogPostProps> = async (context) => {
  const { slug } = context.params!;

  const post = await fetch(`https://api.example.com/posts/${slug}`)
    .then(r => r.json());

  if (!post) {
    return {
      notFound: true,
      revalidate: 60, // Recheck in 60 seconds
    };
  }

  return {
    props: {
      post,
      revalidatedAt: Date.now(),
    },
    // Revalidate at most once every 60 seconds
    revalidate: 60,
  };
};

export default function BlogPost({ post, revalidatedAt }: BlogPostProps) {
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
      <footer>
        <p>Last updated: {post.updatedAt}</p>
        <p className="text-xs text-gray-500">
          Page regenerated: {new Date(revalidatedAt).toLocaleString()}
        </p>
      </footer>
    </article>
  );
}
```

### On-Demand Revalidation

```typescript
// pages/api/revalidate.ts
import { NextApiRequest, NextApiResponse } from 'next';

interface RevalidateRequest {
  paths?: string[];
  secret?: string;
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Verify secret token
  if (req.query.secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  try {
    const { paths } = req.body as RevalidateRequest;

    if (!paths || paths.length === 0) {
      return res.status(400).json({ message: 'No paths provided' });
    }

    // Revalidate specified paths
    await Promise.all(
      paths.map(path => res.revalidate(path))
    );

    return res.json({
      revalidated: true,
      paths,
      timestamp: Date.now(),
    });
  } catch (err) {
    return res.status(500).json({
      message: 'Error revalidating',
      error: err.message,
    });
  }
}
```

### Webhook-Triggered Revalidation

```typescript
// pages/api/webhooks/content-update.ts
import { NextApiRequest, NextApiResponse } from 'next';
import crypto from 'crypto';

interface WebhookPayload {
  entity: string;
  action: string;
  slug?: string;
  id?: string;
}

function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const hmac = crypto.createHmac('sha256', secret);
  const digest = hmac.update(payload).digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(digest)
  );
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' });
  }

  // Verify webhook signature
  const signature = req.headers['x-webhook-signature'] as string;
  const payload = JSON.stringify(req.body);
  
  if (!verifyWebhookSignature(
    payload,
    signature,
    process.env.WEBHOOK_SECRET!
  )) {
    return res.status(401).json({ message: 'Invalid signature' });
  }

  const webhookData: WebhookPayload = req.body;
  const pathsToRevalidate: string[] = [];

  try {
    // Determine which paths to revalidate
    switch (webhookData.entity) {
      case 'post':
        if (webhookData.slug) {
          pathsToRevalidate.push(`/blog/${webhookData.slug}`);
          pathsToRevalidate.push('/blog'); // Blog listing page
        }
        break;

      case 'category':
        // Revalidate category page and all posts
        pathsToRevalidate.push(`/category/${webhookData.slug}`);
        const posts = await fetchPostsByCategory(webhookData.id!);
        posts.forEach(post => {
          pathsToRevalidate.push(`/blog/${post.slug}`);
        });
        break;

      case 'author':
        // Revalidate author page
        pathsToRevalidate.push(`/author/${webhookData.slug}`);
        break;

      default:
        return res.status(400).json({ message: 'Unknown entity type' });
    }

    // Revalidate all affected paths
    const results = await Promise.allSettled(
      pathsToRevalidate.map(path => res.revalidate(path))
    );

    const successful = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;

    return res.json({
      revalidated: true,
      successful,
      failed,
      paths: pathsToRevalidate,
      timestamp: Date.now(),
    });
  } catch (err) {
    console.error('Revalidation error:', err);
    return res.status(500).json({
      message: 'Error during revalidation',
      error: err.message,
    });
  }
}

async function fetchPostsByCategory(categoryId: string): Promise<Post[]> {
  const response = await fetch(
    `https://api.example.com/posts?category=${categoryId}`
  );
  return response.json();
}
```

## Advanced ISR Patterns

### Stale-While-Revalidate

```typescript
// pages/products/[id].tsx
import { GetStaticProps, GetStaticPaths } from 'next';
import { useState, useEffect } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
  lastUpdated: string;
}

interface ProductPageProps {
  product: Product;
  generatedAt: number;
}

export const getStaticPaths: GetStaticPaths = async () => {
  const topProducts = await fetchTopProducts(50);

  return {
    paths: topProducts.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking',
  };
};

export const getStaticProps: GetStaticProps<ProductPageProps> = async (context) => {
  const { id } = context.params!;
  
  const product = await fetch(`https://api.example.com/products/${id}`)
    .then(r => r.json());

  return {
    props: {
      product,
      generatedAt: Date.now(),
    },
    // Revalidate every 30 seconds
    revalidate: 30,
  };
};

export default function ProductPage({ product, generatedAt }: ProductPageProps) {
  const [currentStock, setCurrentStock] = useState(product.stock);
  const [isStale, setIsStale] = useState(false);

  useEffect(() => {
    // Check if data is stale (older than 1 minute)
    const age = Date.now() - generatedAt;
    setIsStale(age > 60000);

    // Fetch fresh stock data client-side for critical info
    async function fetchFreshStock() {
      try {
        const response = await fetch(`/api/products/${product.id}/stock`);
        const data = await response.json();
        setCurrentStock(data.stock);
      } catch (error) {
        console.error('Failed to fetch fresh stock:', error);
      }
    }

    if (isStale) {
      fetchFreshStock();
    }
  }, [product.id, generatedAt]);

  return (
    <div>
      <h1>{product.name}</h1>
      <p className="text-2xl">${product.price}</p>
      
      <div className="stock-info">
        <p>Stock: {currentStock}</p>
        {isStale && (
          <span className="badge">Live data</span>
        )}
      </div>

      <button disabled={currentStock === 0}>
        {currentStock > 0 ? 'Add to Cart' : 'Out of Stock'}
      </button>
    </div>
  );
}
```

### Conditional Revalidation

```typescript
// lib/smart-revalidation.ts
interface RevalidationStrategy {
  path: string;
  priority: 'high' | 'medium' | 'low';
  lastRevalidated?: number;
  revalidateInterval: number;
}

class SmartRevalidationManager {
  private strategies: Map<string, RevalidationStrategy> = new Map();

  constructor() {
    // Define revalidation strategies
    this.addStrategy('/blog/*', 'medium', 300); // 5 minutes
    this.addStrategy('/products/*', 'high', 60); // 1 minute
    this.addStrategy('/docs/*', 'low', 3600); // 1 hour
  }

  addStrategy(
    pathPattern: string,
    priority: 'high' | 'medium' | 'low',
    interval: number
  ): void {
    this.strategies.set(pathPattern, {
      path: pathPattern,
      priority,
      revalidateInterval: interval,
    });
  }

  shouldRevalidate(path: string): boolean {
    const strategy = this.findStrategy(path);
    if (!strategy) return false;

    if (!strategy.lastRevalidated) return true;

    const elapsed = Date.now() - strategy.lastRevalidated;
    return elapsed >= strategy.revalidateInterval * 1000;
  }

  async revalidate(path: string): Promise<boolean> {
    if (!this.shouldRevalidate(path)) {
      console.log(`Skipping revalidation for ${path} (too recent)`);
      return false;
    }

    try {
      const response = await fetch('/api/revalidate', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': process.env.REVALIDATE_SECRET!,
        },
        body: JSON.stringify({ paths: [path] }),
      });

      if (response.ok) {
        const strategy = this.findStrategy(path);
        if (strategy) {
          strategy.lastRevalidated = Date.now();
        }
        return true;
      }

      return false;
    } catch (error) {
      console.error(`Revalidation failed for ${path}:`, error);
      return false;
    }
  }

  private findStrategy(path: string): RevalidationStrategy | undefined {
    for (const [pattern, strategy] of this.strategies) {
      if (this.matchesPattern(path, pattern)) {
        return strategy;
      }
    }
    return undefined;
  }

  private matchesPattern(path: string, pattern: string): boolean {
    const regex = new RegExp(
      '^' + pattern.replace(/\*/g, '.*').replace(/\?/g, '.') + '$'
    );
    return regex.test(path);
  }
}

export const revalidationManager = new SmartRevalidationManager();
```

### Batch Revalidation

```typescript
// pages/api/revalidate-batch.ts
import { NextApiRequest, NextApiResponse } from 'next';

interface BatchRevalidateRequest {
  patterns?: string[];
  entities?: Array<{
    type: string;
    id: string;
  }>;
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' });
  }

  // Verify authentication
  const apiKey = req.headers['x-api-key'];
  if (apiKey !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  const { patterns, entities } = req.body as BatchRevalidateRequest;
  const pathsToRevalidate: Set<string> = new Set();

  try {
    // Handle pattern-based revalidation
    if (patterns) {
      for (const pattern of patterns) {
        const paths = await expandPattern(pattern);
        paths.forEach(path => pathsToRevalidate.add(path));
      }
    }

    // Handle entity-based revalidation
    if (entities) {
      for (const entity of entities) {
        const paths = await getPathsForEntity(entity.type, entity.id);
        paths.forEach(path => pathsToRevalidate.add(path));
      }
    }

    // Revalidate in batches of 10
    const allPaths = Array.from(pathsToRevalidate);
    const batchSize = 10;
    const results = [];

    for (let i = 0; i < allPaths.length; i += batchSize) {
      const batch = allPaths.slice(i, i + batchSize);
      const batchResults = await Promise.allSettled(
        batch.map(path => res.revalidate(path))
      );
      results.push(...batchResults);

      // Small delay between batches to avoid overwhelming
      if (i + batchSize < allPaths.length) {
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    }

    const successful = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;

    return res.json({
      success: true,
      total: allPaths.length,
      successful,
      failed,
      paths: allPaths,
    });
  } catch (error) {
    console.error('Batch revalidation error:', error);
    return res.status(500).json({
      message: 'Batch revalidation failed',
      error: error.message,
    });
  }
}

async function expandPattern(pattern: string): Promise<string[]> {
  // Implementation depends on your routing structure
  // Example: /blog/* -> fetch all blog post paths
  if (pattern === '/blog/*') {
    const posts = await fetch('https://api.example.com/posts')
      .then(r => r.json());
    return posts.map((p: Post) => `/blog/${p.slug}`);
  }
  
  return [pattern];
}

async function getPathsForEntity(type: string, id: string): Promise<string[]> {
  const paths: string[] = [];

  switch (type) {
    case 'author':
      // Author page + all their posts
      const author = await fetchAuthor(id);
      paths.push(`/author/${author.slug}`);
      const posts = await fetchAuthorPosts(id);
      posts.forEach(post => paths.push(`/blog/${post.slug}`));
      break;

    case 'category':
      // Category page + all posts in category
      const category = await fetchCategory(id);
      paths.push(`/category/${category.slug}`);
      const categoryPosts = await fetchCategoryPosts(id);
      categoryPosts.forEach(post => paths.push(`/blog/${post.slug}`));
      break;

    default:
      throw new Error(`Unknown entity type: ${type}`);
  }

  return paths;
}
```

## Angular Universal ISR (Experimental)

### Angular ISR Setup

```typescript
// server.ts
import 'zone.js/node';
import { ngExpressEngine } from '@nguniversal/express-engine';
import { ApplicationRef } from '@angular/core';
import { first } from 'rxjs/operators';
import express from 'express';
import { existsSync, mkdirSync, writeFileSync, readFileSync } from 'fs';
import { join } from 'path';

const app = express();
const distFolder = join(process.cwd(), 'dist/my-app/browser');
const cacheFolder = join(process.cwd(), '.cache');

// Ensure cache directory exists
if (!existsSync(cacheFolder)) {
  mkdirSync(cacheFolder, { recursive: true });
}

interface CacheEntry {
  html: string;
  timestamp: number;
  revalidate: number;
}

class ISRCache {
  private cache: Map<string, CacheEntry> = new Map();

  get(path: string): CacheEntry | null {
    const cachePath = join(cacheFolder, `${this.hashPath(path)}.json`);
    
    if (existsSync(cachePath)) {
      const data = readFileSync(cachePath, 'utf-8');
      return JSON.parse(data);
    }

    return this.cache.get(path) || null;
  }

  set(path: string, html: string, revalidate: number): void {
    const entry: CacheEntry = {
      html,
      timestamp: Date.now(),
      revalidate,
    };

    // Memory cache
    this.cache.set(path, entry);

    // Disk cache
    const cachePath = join(cacheFolder, `${this.hashPath(path)}.json`);
    writeFileSync(cachePath, JSON.stringify(entry));
  }

  shouldRevalidate(entry: CacheEntry): boolean {
    const age = Date.now() - entry.timestamp;
    return age >= entry.revalidate * 1000;
  }

  private hashPath(path: string): string {
    return Buffer.from(path).toString('base64').replace(/[/+=]/g, '_');
  }
}

const cache = new ISRCache();

app.engine('html', ngExpressEngine({
  bootstrap: AppServerModule,
}));

app.set('view engine', 'html');
app.set('views', distFolder);

// ISR middleware
app.get('*', async (req, res, next) => {
  const path = req.path;
  
  // Check cache
  const cached = cache.get(path);

  if (cached) {
    // Serve cached content immediately
    res.send(cached.html);

    // Background revalidation if needed
    if (cache.shouldRevalidate(cached)) {
      regeneratePage(path).catch(err => {
        console.error(`Background revalidation failed for ${path}:`, err);
      });
    }

    return;
  }

  // No cache, render now
  next();
});

// Regular SSR
app.get('*', (req, res) => {
  res.render('index', { req }, (err, html) => {
    if (err) {
      return res.status(500).send('Rendering error');
    }

    // Cache the rendered HTML
    cache.set(req.path, html, 60); // 60 second revalidation

    res.send(html);
  });
});

async function regeneratePage(path: string): Promise<void> {
  // Render page in background
  const html = await new Promise<string>((resolve, reject) => {
    app.render('index', { req: { path } }, (err, html) => {
      if (err) reject(err);
      else resolve(html);
    });
  });

  // Update cache
  cache.set(path, html, 60);
}

const port = process.env.PORT || 4000;
app.listen(port, () => {
  console.log(`Angular Universal with ISR listening on port ${port}`);
});
```

## Monitoring and Analytics

### ISR Performance Tracking

```typescript
// lib/isr-analytics.ts
interface ISRMetrics {
  path: string;
  cacheHit: boolean;
  regenerated: boolean;
  renderTime?: number;
  timestamp: number;
}

class ISRAnalytics {
  private metrics: ISRMetrics[] = [];

  recordRequest(metrics: ISRMetrics): void {
    this.metrics.push(metrics);

    // Send to analytics service
    this.sendToAnalytics(metrics);

    // Keep only last 1000 metrics in memory
    if (this.metrics.length > 1000) {
      this.metrics.shift();
    }
  }

  getStats(path?: string) {
    const relevantMetrics = path
      ? this.metrics.filter(m => m.path === path)
      : this.metrics;

    const total = relevantMetrics.length;
    const cacheHits = relevantMetrics.filter(m => m.cacheHit).length;
    const regenerations = relevantMetrics.filter(m => m.regenerated).length;
    const avgRenderTime = relevantMetrics
      .filter(m => m.renderTime)
      .reduce((sum, m) => sum + m.renderTime!, 0) / total;

    return {
      total,
      cacheHits,
      cacheHitRate: cacheHits / total,
      regenerations,
      regenerationRate: regenerations / total,
      avgRenderTime,
    };
  }

  private sendToAnalytics(metrics: ISRMetrics): void {
    // Send to your analytics service
    if (process.env.ANALYTICS_ENDPOINT) {
      fetch(process.env.ANALYTICS_ENDPOINT, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(metrics),
      }).catch(err => {
        console.error('Failed to send analytics:', err);
      });
    }
  }
}

export const isrAnalytics = new ISRAnalytics();
```

### Health Check Endpoint

```typescript
// pages/api/health/isr.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { isrAnalytics } from '../../../lib/isr-analytics';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const stats = isrAnalytics.getStats();

  const health = {
    status: 'healthy',
    timestamp: Date.now(),
    metrics: stats,
    warnings: [],
  };

  // Check for issues
  if (stats.cacheHitRate < 0.5) {
    health.warnings.push('Low cache hit rate (<50%)');
    health.status = 'degraded';
  }

  if (stats.avgRenderTime > 1000) {
    health.warnings.push('High average render time (>1s)');
    health.status = 'degraded';
  }

  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
}
```

## Common Mistakes

### 1. Setting Revalidation Too Low

```typescript
// BAD: Revalidating too frequently
export const getStaticProps: GetStaticProps = async () => {
  const data = await fetchData();
  return {
    props: { data },
    revalidate: 1, // Too frequent! Heavy load
  };
};

// GOOD: Reasonable revalidation interval
export const getStaticProps: GetStaticProps = async () => {
  const data = await fetchData();
  return {
    props: { data },
    revalidate: 60, // 1 minute is usually sufficient
  };
};
```

### 2. Not Handling Revalidation Failures

```typescript
// BAD: No error handling
await res.revalidate('/some-path');

// GOOD: Handle failures
try {
  await res.revalidate('/some-path');
  console.log('Revalidated successfully');
} catch (error) {
  console.error('Revalidation failed:', error);
  // Fallback: trigger rebuild or alert
  await triggerManualRebuild('/some-path');
}
```

### 3. Over-Revalidating Related Pages

```typescript
// BAD: Revalidating everything on every change
await Promise.all([
  res.revalidate('/'),
  res.revalidate('/blog'),
  res.revalidate('/about'),
  res.revalidate('/contact'),
  // ... revalidating entire site!
]);

// GOOD: Only revalidate affected pages
const affectedPaths = determineAffectedPaths(changedEntity);
await Promise.all(
  affectedPaths.map(path => res.revalidate(path))
);
```

## Best Practices

### 1. Use Different Revalidation Intervals

```typescript
// Critical data: short interval
export const getStaticProps: GetStaticProps = async () => {
  const stockData = await fetchStockData();
  return {
    props: { stockData },
    revalidate: 30, // 30 seconds for time-sensitive data
  };
};

// Regular content: medium interval
export const getStaticProps: GetStaticProps = async () => {
  const blogPost = await fetchBlogPost();
  return {
    props: { blogPost },
    revalidate: 300, // 5 minutes for regular content
  };
};

// Static content: long interval
export const getStaticProps: GetStaticProps = async () => {
  const documentation = await fetchDocs();
  return {
    props: { documentation },
    revalidate: 3600, // 1 hour for rarely changing content
  };
};
```

### 2. Implement Graceful Degradation

```typescript
export const getStaticProps: GetStaticProps = async (context) => {
  try {
    const data = await fetchData();
    return {
      props: { data, error: null },
      revalidate: 60,
    };
  } catch (error) {
    console.error('Data fetch failed:', error);
    
    // Return cached/fallback data
    return {
      props: {
        data: getFallbackData(),
        error: 'Using cached data',
      },
      revalidate: 10, // Retry sooner
    };
  }
};
```

### 3. Monitor Revalidation Performance

```typescript
// Track revalidation metrics
export async function revalidateWithMetrics(
  res: NextApiResponse,
  path: string
): Promise<boolean> {
  const startTime = Date.now();
  
  try {
    await res.revalidate(path);
    const duration = Date.now() - startTime;
    
    // Log metrics
    console.log(`Revalidated ${path} in ${duration}ms`);
    
    // Send to monitoring
    await sendMetric('revalidation.success', {
      path,
      duration,
    });
    
    return true;
  } catch (error) {
    const duration = Date.now() - startTime;
    
    console.error(`Revalidation failed for ${path}:`, error);
    
    await sendMetric('revalidation.failure', {
      path,
      duration,
      error: error.message,
    });
    
    return false;
  }
}
```

## When to Use ISR

### Ideal Use Cases

1. **E-commerce product pages** - Prices and stock change periodically
2. **News websites** - New articles published throughout day
3. **Social content** - User-generated content with moderate updates
4. **Dashboard pages** - Metrics updated at regular intervals
5. **Content-heavy sites** - Large sites with occasional updates

### Not Recommended For

1. **Real-time data** - Use SSR or client-side fetching
2. **Highly personalized content** - ISR doesn't support per-user caching
3. **Authentication-required pages** - Static pages can't be personalized
4. **Constantly changing data** - Use SSR instead

## Interview Questions

1. **What is the difference between ISR and SSG?**
   - ISR allows updating static pages without full rebuild
   - ISR uses stale-while-revalidate pattern
   - ISR provides balance between SSG performance and SSR freshness

2. **How does ISR revalidation work?**
   - First user gets stale content immediately
   - Background process regenerates page if revalidate interval passed
   - Subsequent users get fresh content
   - Process repeats on next interval

3. **What is the trade-off between revalidate intervals?**
   - Shorter intervals: fresher content, more server load
   - Longer intervals: less load, potentially stale content
   - Choose based on content update frequency

4. **How do you trigger on-demand revalidation?**
   - Create API route with res.revalidate()
   - Protect with secret token
   - Call from CMS webhooks or manual triggers
   - Handle multiple paths in single call

5. **What happens if revalidation fails?**
   - Old cached version continues to serve
   - Next request attempts revalidation again
   - Implement retry logic and monitoring
   - Consider fallback to SSR if critical

## Key Takeaways

1. ISR combines static generation benefits with content freshness
2. Serves stale content immediately while regenerating in background
3. Use time-based revalidation for periodic updates
4. Use on-demand revalidation for immediate updates
5. Choose revalidate intervals based on content update frequency
6. Implement proper error handling for revalidation failures
7. Monitor revalidation performance and success rates
8. Use fallback strategies for critical data
9. Consider CDN cache behavior with ISR
10. Not suitable for real-time or personalized content

## Resources

- [Next.js ISR Documentation](https://nextjs.org/docs/basic-features/data-fetching/incremental-static-regeneration)
- [Vercel ISR Guide](https://vercel.com/docs/concepts/incremental-static-regeneration)
- [ISR Best Practices](https://www.patterns.dev/posts/incremental-static-rendering)
- [On-Demand ISR](https://nextjs.org/docs/basic-features/data-fetching/incremental-static-regeneration#on-demand-revalidation)
- [ISR Performance](https://web.dev/rendering-on-the-web/)
