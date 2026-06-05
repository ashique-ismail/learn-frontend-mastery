# Static Site Generation (SSG)

## The Idea

**In plain English:** Static Site Generation is when a website builds all its pages ahead of time — before anyone visits — and stores them as ready-to-go files, so any visitor just picks up the finished page instantly. Think of "building" as doing all the cooking in advance, and "static" means the food is already plated and sitting on the counter, not cooked fresh to order.

**Real-world analogy:** A school prints 500 copies of a class newsletter at the start of the month, and stacks them by the door. Any student who walks in just grabs one — no waiting, no printing on demand.

- The printing press running at the start of the month = the build process that generates all HTML files
- Each printed copy of the newsletter = a pre-built HTML page stored on a server
- The stack of copies by the door = the CDN (Content Delivery Network) that hands pages out instantly
- A student grabbing a copy = a visitor's browser receiving the page with zero server processing

---

## Overview

Static Site Generation (SSG) is a rendering strategy where HTML pages are generated at build time rather than request time. The pre-rendered HTML is then served directly from a CDN, providing exceptional performance and scalability. SSG is ideal for content that doesn't change frequently or can be regenerated periodically.

## How SSG Works

```
Build Time Flow:
┌─────────────────────────────────────────────────────────────┐
│                    BUILD PROCESS                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Data Fetching        2. Page Generation                 │
│  ┌─────────────┐        ┌──────────────┐                   │
│  │   CMS API   │───────▶│ HTML Builder │                   │
│  │ Database    │        │              │                   │
│  │ Files       │        │ React/Angular│                   │
│  └─────────────┘        └──────┬───────┘                   │
│                                 │                            │
│                                 ▼                            │
│                         ┌──────────────┐                    │
│                         │  Static HTML │                    │
│                         │   + Assets   │                    │
│                         └──────┬───────┘                    │
│                                 │                            │
│                                 ▼                            │
│                         ┌──────────────┐                    │
│                         │   Deploy to  │                    │
│                         │     CDN      │                    │
│                         └──────────────┘                    │
└─────────────────────────────────────────────────────────────┘

Request Time Flow:
┌──────────┐         ┌─────────┐         ┌──────────┐
│  Client  │────────▶│   CDN   │────────▶│  Static  │
│  Browser │◀────────│  Edge   │◀────────│   HTML   │
└──────────┘         └─────────┘         └──────────┘
   Fast Response (< 50ms from CDN)
```

## Next.js Implementation

### Basic Static Generation

```typescript
// pages/blog/[slug].tsx
import { GetStaticPaths, GetStaticProps } from 'next';

interface Post {
  slug: string;
  title: string;
  content: string;
  author: string;
  publishedAt: string;
}

interface BlogPostProps {
  post: Post;
}

// Generate static paths at build time
export const getStaticPaths: GetStaticPaths = async () => {
  // Fetch all blog posts
  const response = await fetch('https://api.example.com/posts');
  const posts = await response.json();

  // Generate paths for all posts
  const paths = posts.map((post: Post) => ({
    params: { slug: post.slug },
  }));

  return {
    paths,
    fallback: false, // 404 for non-existent paths
  };
};

// Generate static props for each page
export const getStaticProps: GetStaticProps<BlogPostProps> = async (context) => {
  const { slug } = context.params!;

  // Fetch post data at build time
  const response = await fetch(`https://api.example.com/posts/${slug}`);
  const post = await response.json();

  return {
    props: {
      post,
    },
    // Optional: Revalidate every 3600 seconds (ISR)
    // revalidate: 3600,
  };
};

// Component receives static props
export default function BlogPost({ post }: BlogPostProps) {
  return (
    <article>
      <h1>{post.title}</h1>
      <p>By {post.author} on {new Date(post.publishedAt).toLocaleDateString()}</p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

### Advanced Static Generation with Multiple Data Sources

```typescript
// pages/products/[category]/[id].tsx
import { GetStaticPaths, GetStaticProps } from 'next';

interface Product {
  id: string;
  category: string;
  name: string;
  description: string;
  price: number;
  reviews: Review[];
  relatedProducts: string[];
}

interface Review {
  id: string;
  rating: number;
  comment: string;
  author: string;
}

interface ProductPageProps {
  product: Product;
  relatedProducts: Product[];
}

export const getStaticPaths: GetStaticPaths = async () => {
  // Fetch all categories
  const categories = await fetch('https://api.example.com/categories').then(r => r.json());
  
  // Fetch all products
  const allProducts = await fetch('https://api.example.com/products').then(r => r.json());

  // Generate paths for all category/product combinations
  const paths = allProducts.map((product: Product) => ({
    params: {
      category: product.category,
      id: product.id,
    },
  }));

  return {
    paths,
    fallback: 'blocking', // Generate on-demand for missing paths
  };
};

export const getStaticProps: GetStaticProps<ProductPageProps> = async (context) => {
  const { category, id } = context.params!;

  // Parallel data fetching
  const [product, reviews, related] = await Promise.all([
    fetch(`https://api.example.com/products/${id}`).then(r => r.json()),
    fetch(`https://api.example.com/products/${id}/reviews`).then(r => r.json()),
    fetch(`https://api.example.com/products/${id}/related`).then(r => r.json()),
  ]);

  // Fetch related product details
  const relatedProducts = await Promise.all(
    related.map((rid: string) =>
      fetch(`https://api.example.com/products/${rid}`).then(r => r.json())
    )
  );

  return {
    props: {
      product: { ...product, reviews },
      relatedProducts,
    },
  };
};

export default function ProductPage({ product, relatedProducts }: ProductPageProps) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      <p>{product.description}</p>
      
      <section>
        <h2>Reviews</h2>
        {product.reviews.map(review => (
          <div key={review.id}>
            <p>Rating: {review.rating}/5</p>
            <p>{review.comment}</p>
            <p>- {review.author}</p>
          </div>
        ))}
      </section>

      <section>
        <h2>Related Products</h2>
        {relatedProducts.map(rp => (
          <div key={rp.id}>
            <h3>{rp.name}</h3>
            <p>${rp.price}</p>
          </div>
        ))}
      </section>
    </div>
  );
}
```

## Angular Universal with Prerendering

### Angular Configuration

```typescript
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:browser",
          "options": {
            "outputPath": "dist/my-app/browser"
          }
        },
        "server": {
          "builder": "@angular-devkit/build-angular:server",
          "options": {
            "outputPath": "dist/my-app/server"
          }
        },
        "prerender": {
          "builder": "@nguniversal/builders:prerender",
          "options": {
            "routes": [
              "/",
              "/about",
              "/contact",
              "/blog/post-1",
              "/blog/post-2"
            ]
          }
        }
      }
    }
  }
}
```

### Dynamic Route Generation

```typescript
// prerender-routes.ts
import { writeFileSync } from 'fs';
import { join } from 'path';

interface BlogPost {
  slug: string;
  title: string;
}

async function generateRoutes() {
  // Fetch dynamic data
  const response = await fetch('https://api.example.com/posts');
  const posts: BlogPost[] = await response.json();

  // Generate route list
  const routes = [
    '/',
    '/about',
    '/contact',
    ...posts.map(post => `/blog/${post.slug}`),
  ];

  // Write to routes file
  const routesPath = join(__dirname, 'routes.txt');
  writeFileSync(routesPath, routes.join('\n'));

  console.log(`Generated ${routes.length} routes for prerendering`);
}

generateRoutes();
```

### Angular Component with Static Data

```typescript
// blog-post.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { TransferState, makeStateKey } from '@angular/platform-browser';

interface Post {
  slug: string;
  title: string;
  content: string;
  author: string;
  publishedAt: string;
}

const POST_KEY = makeStateKey<Post>('post');

@Component({
  selector: 'app-blog-post',
  template: `
    <article *ngIf="post">
      <h1>{{ post.title }}</h1>
      <p>By {{ post.author }} on {{ post.publishedAt | date }}</p>
      <div [innerHTML]="post.content"></div>
    </article>
  `
})
export class BlogPostComponent implements OnInit {
  post: Post | null = null;

  constructor(
    private route: ActivatedRoute,
    private transferState: TransferState
  ) {}

  ngOnInit(): void {
    // Check if data is available from SSG
    const cachedPost = this.transferState.get(POST_KEY, null);
    
    if (cachedPost) {
      this.post = cachedPost;
      // Remove from transfer state to prevent memory leaks
      this.transferState.remove(POST_KEY);
    } else {
      // Fallback: fetch data client-side
      this.loadPost();
    }
  }

  private async loadPost(): Promise<void> {
    const slug = this.route.snapshot.paramMap.get('slug');
    const response = await fetch(`https://api.example.com/posts/${slug}`);
    this.post = await response.json();
  }
}
```

### Server-Side Data Fetching for SSG

```typescript
// server.ts
import 'zone.js/node';
import { ngExpressEngine } from '@nguniversal/express-engine';
import { APP_BASE_HREF } from '@angular/common';
import { existsSync } from 'fs';
import { join } from 'path';
import express from 'express';

const app = express();
const distFolder = join(process.cwd(), 'dist/my-app/browser');

app.engine('html', ngExpressEngine({
  bootstrap: AppServerModule,
  providers: [
    {
      provide: 'REQUEST',
      useFactory: () => {
        // Provide request object for SSG
        return {};
      }
    }
  ]
}));

app.set('view engine', 'html');
app.set('views', distFolder);

// Serve static files
app.get('*.*', express.static(distFolder, {
  maxAge: '1y'
}));

// All regular routes use the Universal engine
app.get('*', (req, res) => {
  res.render('index', {
    req,
    providers: [
      { provide: APP_BASE_HREF, useValue: req.baseUrl }
    ]
  });
});
```

## Gatsby Static Generation

### GraphQL Data Layer

```typescript
// gatsby-node.ts
import { GatsbyNode } from 'gatsby';
import path from 'path';

interface BlogPost {
  id: string;
  slug: string;
  title: string;
}

export const createPages: GatsbyNode['createPages'] = async ({ graphql, actions }) => {
  const { createPage } = actions;

  // Query all blog posts
  const result = await graphql<{
    allMarkdownRemark: {
      nodes: BlogPost[];
    };
  }>`
    query {
      allMarkdownRemark {
        nodes {
          id
          frontmatter {
            slug
            title
          }
        }
      }
    }
  `;

  if (result.errors || !result.data) {
    throw new Error('Failed to fetch posts');
  }

  // Create pages for each post
  result.data.allMarkdownRemark.nodes.forEach((node) => {
    createPage({
      path: `/blog/${node.frontmatter.slug}`,
      component: path.resolve('./src/templates/blog-post.tsx'),
      context: {
        id: node.id,
        slug: node.frontmatter.slug,
      },
    });
  });
};
```

### Gatsby Template Component

```typescript
// src/templates/blog-post.tsx
import React from 'react';
import { graphql, PageProps } from 'gatsby';

interface BlogPostData {
  markdownRemark: {
    id: string;
    html: string;
    frontmatter: {
      title: string;
      date: string;
      author: string;
    };
  };
}

export const query = graphql`
  query BlogPostBySlug($id: String!) {
    markdownRemark(id: { eq: $id }) {
      id
      html
      frontmatter {
        title
        date(formatString: "MMMM DD, YYYY")
        author
      }
    }
  }
`;

const BlogPostTemplate: React.FC<PageProps<BlogPostData>> = ({ data }) => {
  const { markdownRemark } = data;
  const { frontmatter, html } = markdownRemark;

  return (
    <article>
      <h1>{frontmatter.title}</h1>
      <p>By {frontmatter.author} on {frontmatter.date}</p>
      <div dangerouslySetInnerHTML={{ __html: html }} />
    </article>
  );
};

export default BlogPostTemplate;
```

## Build-Time Data Fetching Patterns

### Caching External API Calls

```typescript
// lib/data-cache.ts
import fs from 'fs';
import path from 'path';
import crypto from 'crypto';

const CACHE_DIR = path.join(process.cwd(), '.cache');

interface CacheOptions {
  ttl?: number; // Time to live in seconds
  forceRefresh?: boolean;
}

export async function cachedFetch<T>(
  url: string,
  options: CacheOptions = {}
): Promise<T> {
  const { ttl = 3600, forceRefresh = false } = options;

  // Create cache key from URL
  const cacheKey = crypto.createHash('md5').update(url).digest('hex');
  const cachePath = path.join(CACHE_DIR, `${cacheKey}.json`);

  // Ensure cache directory exists
  if (!fs.existsSync(CACHE_DIR)) {
    fs.mkdirSync(CACHE_DIR, { recursive: true });
  }

  // Check cache
  if (!forceRefresh && fs.existsSync(cachePath)) {
    const stats = fs.statSync(cachePath);
    const age = (Date.now() - stats.mtimeMs) / 1000;

    if (age < ttl) {
      console.log(`Cache hit: ${url}`);
      const cached = fs.readFileSync(cachePath, 'utf-8');
      return JSON.parse(cached);
    }
  }

  // Fetch fresh data
  console.log(`Fetching: ${url}`);
  const response = await fetch(url);
  const data = await response.json();

  // Write to cache
  fs.writeFileSync(cachePath, JSON.stringify(data, null, 2));

  return data;
}
```

### Parallel Data Fetching

```typescript
// lib/data-loader.ts
import { cachedFetch } from './data-cache';

interface SiteData {
  posts: Post[];
  categories: Category[];
  authors: Author[];
  tags: Tag[];
}

export async function loadAllSiteData(): Promise<SiteData> {
  console.log('Loading all site data...');

  // Fetch all data in parallel
  const [posts, categories, authors, tags] = await Promise.all([
    cachedFetch<Post[]>('https://api.example.com/posts'),
    cachedFetch<Category[]>('https://api.example.com/categories'),
    cachedFetch<Author[]>('https://api.example.com/authors'),
    cachedFetch<Tag[]>('https://api.example.com/tags'),
  ]);

  console.log(`Loaded ${posts.length} posts, ${categories.length} categories`);

  return { posts, categories, authors, tags };
}

// Batch data fetching with rate limiting
export async function fetchBatch<T>(
  urls: string[],
  concurrency: number = 5
): Promise<T[]> {
  const results: T[] = [];
  
  for (let i = 0; i < urls.length; i += concurrency) {
    const batch = urls.slice(i, i + concurrency);
    const batchResults = await Promise.all(
      batch.map(url => cachedFetch<T>(url))
    );
    results.push(...batchResults);
    
    // Log progress
    console.log(`Processed ${Math.min(i + concurrency, urls.length)}/${urls.length} items`);
  }

  return results;
}
```

## Incremental Builds

### Next.js with Turbopack

```typescript
// next.config.js
module.exports = {
  experimental: {
    turbo: {
      // Enable incremental builds
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js',
        },
      },
    },
  },
  
  // Configure build cache
  onDemandEntries: {
    maxInactiveAge: 25 * 1000,
    pagesBufferLength: 2,
  },

  // Webpack cache configuration
  webpack: (config, { isServer }) => {
    config.cache = {
      type: 'filesystem',
      buildDependencies: {
        config: [__filename],
      },
    };
    return config;
  },
};
```

### Gatsby Incremental Builds

```typescript
// gatsby-config.ts
import type { GatsbyConfig } from 'gatsby';

const config: GatsbyConfig = {
  flags: {
    FAST_DEV: true,
    DEV_SSR: false,
    PRESERVE_FILE_DOWNLOAD_CACHE: true,
    PARALLEL_SOURCING: true,
  },
  
  plugins: [
    {
      resolve: 'gatsby-plugin-sharp',
      options: {
        defaults: {
          formats: ['auto', 'webp', 'avif'],
          placeholder: 'blurred',
          quality: 80,
        },
      },
    },
  ],
};

export default config;
```

## Regeneration Strategies

### Manual Rebuild Triggers

```typescript
// scripts/rebuild.ts
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

interface RebuildOptions {
  pages?: string[];
  clearCache?: boolean;
}

async function rebuildSite(options: RebuildOptions = {}): Promise<void> {
  const { pages, clearCache = false } = options;

  try {
    // Clear cache if requested
    if (clearCache) {
      console.log('Clearing build cache...');
      await execAsync('rm -rf .next/cache');
    }

    // Rebuild specific pages or entire site
    if (pages && pages.length > 0) {
      console.log(`Rebuilding pages: ${pages.join(', ')}`);
      // Trigger incremental build (implementation depends on framework)
      for (const page of pages) {
        await execAsync(`next build --page ${page}`);
      }
    } else {
      console.log('Rebuilding entire site...');
      await execAsync('next build');
    }

    console.log('Rebuild complete!');
  } catch (error) {
    console.error('Rebuild failed:', error);
    throw error;
  }
}

// Webhook handler for CMS updates
export async function handleWebhook(payload: any): Promise<void> {
  const { entity, slug } = payload;

  switch (entity) {
    case 'post':
      await rebuildSite({ pages: [`/blog/${slug}`] });
      break;
    case 'category':
      // Rebuild all posts in category
      await rebuildSite({ clearCache: true });
      break;
    default:
      console.log(`Unknown entity: ${entity}`);
  }
}
```

### Scheduled Regeneration

```typescript
// scripts/scheduled-rebuild.ts
import cron from 'node-cron';
import { rebuildSite } from './rebuild';

// Rebuild every day at 2 AM
cron.schedule('0 2 * * *', async () => {
  console.log('Starting scheduled rebuild...');
  try {
    await rebuildSite({ clearCache: true });
    console.log('Scheduled rebuild complete');
  } catch (error) {
    console.error('Scheduled rebuild failed:', error);
    // Send alert notification
    await sendAlert('Rebuild failed', error);
  }
});

async function sendAlert(subject: string, error: any): Promise<void> {
  // Implementation depends on notification service
  console.error(`ALERT: ${subject}`, error);
}
```

## Deployment Strategies

### CDN Deployment

```typescript
// scripts/deploy-to-cdn.ts
import AWS from 'aws-sdk';
import fs from 'fs';
import path from 'path';
import mime from 'mime-types';

const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
});

interface DeployOptions {
  bucket: string;
  directory: string;
  cacheControl?: string;
}

async function deployToS3(options: DeployOptions): Promise<void> {
  const { bucket, directory, cacheControl = 'public, max-age=31536000' } = options;

  const files = getAllFiles(directory);

  console.log(`Deploying ${files.length} files to S3...`);

  for (const file of files) {
    const fileContent = fs.readFileSync(file);
    const key = path.relative(directory, file);
    const contentType = mime.lookup(file) || 'application/octet-stream';

    await s3.putObject({
      Bucket: bucket,
      Key: key,
      Body: fileContent,
      ContentType: contentType,
      CacheControl: cacheControl,
    }).promise();

    console.log(`Uploaded: ${key}`);
  }

  console.log('Deployment complete!');
}

function getAllFiles(dir: string): string[] {
  const files: string[] = [];
  const items = fs.readdirSync(dir);

  for (const item of items) {
    const fullPath = path.join(dir, item);
    if (fs.statSync(fullPath).isDirectory()) {
      files.push(...getAllFiles(fullPath));
    } else {
      files.push(fullPath);
    }
  }

  return files;
}

// Deploy build output
deployToS3({
  bucket: 'my-static-site',
  directory: './out',
  cacheControl: 'public, max-age=31536000, immutable',
});
```

### Atomic Deployments

```typescript
// scripts/atomic-deploy.ts
interface AtomicDeployOptions {
  buildId: string;
  sourceDir: string;
  targetBucket: string;
}

async function atomicDeploy(options: AtomicDeployOptions): Promise<void> {
  const { buildId, sourceDir, targetBucket } = options;

  // 1. Upload new version to versioned path
  console.log(`Uploading build ${buildId}...`);
  await uploadToBucket(sourceDir, `${targetBucket}/releases/${buildId}`);

  // 2. Update pointer file atomically
  console.log('Updating current version pointer...');
  await updateVersionPointer(targetBucket, buildId);

  // 3. Invalidate CDN cache
  console.log('Invalidating CDN cache...');
  await invalidateCache();

  // 4. Clean up old versions (keep last 5)
  console.log('Cleaning up old versions...');
  await cleanupOldVersions(targetBucket, 5);

  console.log('Atomic deployment complete!');
}

async function updateVersionPointer(bucket: string, buildId: string): Promise<void> {
  await s3.putObject({
    Bucket: bucket,
    Key: 'current-version.json',
    Body: JSON.stringify({ version: buildId, timestamp: Date.now() }),
    ContentType: 'application/json',
    CacheControl: 'no-cache',
  }).promise();
}
```

## Common Mistakes

### 1. Over-Generating Pages

```typescript
// BAD: Generating thousands of pages
export const getStaticPaths: GetStaticPaths = async () => {
  // Fetching 10,000+ products
  const products = await fetchAllProducts(); // Too many!

  const paths = products.map(p => ({ params: { id: p.id } }));

  return {
    paths, // This will take hours to build
    fallback: false,
  };
};

// GOOD: Use fallback for large datasets
export const getStaticPaths: GetStaticPaths = async () => {
  // Only pre-render top 100 products
  const topProducts = await fetchTopProducts(100);

  const paths = topProducts.map(p => ({ params: { id: p.id } }));

  return {
    paths,
    fallback: 'blocking', // Generate others on-demand
  };
};
```

### 2. Not Handling Build Failures

```typescript
// BAD: No error handling
export const getStaticProps: GetStaticProps = async () => {
  const data = await fetch('https://api.example.com/data').then(r => r.json());
  return { props: { data } };
};

// GOOD: Handle failures gracefully
export const getStaticProps: GetStaticProps = async () => {
  try {
    const response = await fetch('https://api.example.com/data');
    
    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    const data = await response.json();
    return { props: { data } };
  } catch (error) {
    console.error('Failed to fetch data:', error);
    
    // Return fallback data or trigger error page
    return {
      notFound: true, // Or return fallback props
    };
  }
};
```

### 3. Ignoring Build Performance

```typescript
// BAD: Sequential fetching
export const getStaticProps: GetStaticProps = async () => {
  const user = await fetchUser();
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);

  return { props: { user, posts, comments } };
};

// GOOD: Parallel fetching
export const getStaticProps: GetStaticProps = async () => {
  const [user, posts] = await Promise.all([
    fetchUser(),
    fetchPosts(),
  ]);

  // Fetch comments for first post
  const comments = await fetchComments(posts[0].id);

  return { props: { user, posts, comments } };
};
```

## Best Practices

### 1. Use Build Caching

```typescript
// next.config.js
module.exports = {
  webpack: (config, { buildId, dev, isServer }) => {
    // Enable persistent caching
    config.cache = {
      type: 'filesystem',
      cacheDirectory: path.join(__dirname, '.next/cache'),
      buildDependencies: {
        config: [__filename],
      },
    };
    return config;
  },
};
```

### 2. Optimize Data Fetching

```typescript
// lib/optimized-fetch.ts
import pLimit from 'p-limit';

const limit = pLimit(10); // Limit concurrent requests

export async function fetchAllWithLimit<T>(
  urls: string[]
): Promise<T[]> {
  const promises = urls.map(url =>
    limit(() => fetch(url).then(r => r.json()))
  );

  return Promise.all(promises);
}
```

### 3. Implement Smart Regeneration

```typescript
// Only regenerate changed pages
export async function regenerateAffectedPages(changedEntity: string): Promise<void> {
  const affectedPages = await determineAffectedPages(changedEntity);
  
  for (const page of affectedPages) {
    await fetch(`https://yoursite.com/api/revalidate?path=${page}`, {
      method: 'POST',
      headers: { 'x-api-key': process.env.REVALIDATE_TOKEN },
    });
  }
}
```

## When to Use SSG

### Ideal Use Cases

1. **Marketing websites** - Content changes infrequently
2. **Documentation sites** - Version-controlled content
3. **Blogs** - Posts don't change after publication
4. **Product catalogs** - Can regenerate periodically
5. **Landing pages** - Static promotional content

### Not Recommended For

1. **User dashboards** - Personalized, real-time data
2. **Social media feeds** - Constantly updating content
3. **Real-time applications** - Live data requirements
4. **Highly personalized content** - User-specific rendering

## Interview Questions

1. **What is the difference between SSG and SSR?**
   - SSG generates HTML at build time, SSR generates at request time
   - SSG pages served from CDN, SSR requires server
   - SSG better performance, SSR more dynamic

2. **How do you handle frequently changing data with SSG?**
   - Use ISR (Incremental Static Regeneration)
   - Implement webhook-triggered rebuilds
   - Combine with client-side fetching
   - Use scheduled regeneration

3. **What are the trade-offs of SSG?**
   - Pros: Fast, scalable, SEO-friendly, low server costs
   - Cons: Build time increases with pages, data can be stale, requires rebuild for updates

4. **How do you optimize SSG build times?**
   - Use build caching
   - Parallelize data fetching
   - Generate only popular pages statically
   - Use incremental builds

5. **What is fallback in Next.js getStaticPaths?**
   - false: 404 for non-generated paths
   - true: Generate on-demand, show loading
   - 'blocking': Generate on-demand, wait for generation

## Key Takeaways

1. SSG provides exceptional performance by pre-rendering at build time
2. Ideal for content that doesn't change frequently
3. Use fallback strategies for large datasets
4. Implement caching to optimize build times
5. Consider ISR for content that needs periodic updates
6. Deploy to CDN for global distribution
7. Handle build failures gracefully
8. Monitor build times and optimize data fetching
9. Use incremental builds for large sites
10. Combine with client-side fetching for dynamic features

## Resources

- [Next.js Static Generation](https://nextjs.org/docs/basic-features/pages#static-generation)
- [Gatsby Build Process](https://www.gatsbyjs.com/docs/conceptual/overview-of-the-gatsby-build-process/)
- [Angular Universal Prerendering](https://angular.io/guide/prerendering)
- [Jamstack Best Practices](https://jamstack.org/best-practices/)
- [Static Site Generation vs Server-Side Rendering](https://vercel.com/blog/nextjs-server-side-rendering-vs-static-generation)
