# Rendering Patterns Comparison

## Overview

Modern web applications have multiple rendering strategies available, each with distinct trade-offs. This guide compares all major rendering patterns - CSR, SSR, SSG, ISR, Streaming SSR, Islands Architecture, Progressive Hydration, Partial Pre-Rendering (PPR), and Edge Rendering - to help you choose the right approach for your application.

## Quick Comparison Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Rendering Strategy Comparison                                          │
├──────────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬────────┤
│   Metric     │ CSR  │ SSR  │ SSG  │ ISR  │Stream│Island│  PPR │  Edge  │
├──────────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼────────┤
│ TTFB         │  ⭐  │  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐  │
│ FCP          │  ⭐  │ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐  │
│ TTI          │  ⭐  │  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│  ⭐⭐   │
│ SEO          │  ⭐  │ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐  │
│ Freshness    │ ⭐⭐⭐│ ⭐⭐⭐│  ⭐  │  ⭐⭐│ ⭐⭐⭐│  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐  │
│ Personalize  │ ⭐⭐⭐│ ⭐⭐⭐│  ⭐  │  ⭐  │ ⭐⭐⭐│  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐  │
│ Scalability  │ ⭐⭐⭐│  ⭐  │ ⭐⭐⭐│ ⭐⭐⭐│  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐  │
│ Build Time   │ ⭐⭐⭐│ ⭐⭐⭐│  ⭐  │  ⭐⭐│ ⭐⭐⭐│ ⭐⭐⭐│  ⭐⭐│ ⭐⭐⭐  │
│ Server Cost  │  ⭐  │  ⭐  │ ⭐⭐⭐│ ⭐⭐⭐│  ⭐⭐│ ⭐⭐⭐│  ⭐⭐│  ⭐⭐   │
│ Complexity   │  ⭐⭐│  ⭐⭐│ ⭐⭐⭐│  ⭐⭐│  ⭐  │  ⭐  │  ⭐  │  ⭐⭐   │
└──────────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴────────┘

Legend: ⭐ = Poor, ⭐⭐ = Good, ⭐⭐⭐ = Excellent
```

## Detailed Comparison

### Client-Side Rendering (CSR)

```
Flow: Browser → JS Bundle → Render → API → Update

Pros:
✓ Rich interactivity
✓ Reduced server load
✓ Easy to build
✓ Great for SPAs

Cons:
✗ Poor initial load
✗ SEO challenges
✗ Large JS bundles
✗ Slow on low-end devices

Best For:
- Web applications
- Dashboards
- Admin panels
- Tools requiring heavy interactivity
```

```typescript
// CSR Example
function App() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/data')
      .then(r => r.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading...</div>;
  return <div>{data.content}</div>;
}
```

### Server-Side Rendering (SSR)

```
Flow: Request → Server Render → HTML → Browser → Hydrate

Pros:
✓ Fast First Contentful Paint
✓ Great SEO
✓ Fresh content
✓ Personalization

Cons:
✗ Server required
✗ Higher latency
✗ Server costs
✗ Full page reloads

Best For:
- E-commerce
- News sites
- Social media
- Personalized content
```

```typescript
// SSR Example (Next.js)
export async function getServerSideProps(context) {
  const data = await fetchData();
  
  return {
    props: { data },
  };
}

export default function Page({ data }) {
  return <div>{data.content}</div>;
}
```

### Static Site Generation (SSG)

```
Flow: Build Time → Generate HTML → CDN → Browser

Pros:
✓ Extremely fast
✓ Low server costs
✓ Great SEO
✓ Highly scalable

Cons:
✗ Build time increases
✗ Content can be stale
✗ No personalization
✗ Requires rebuild

Best For:
- Blogs
- Documentation
- Marketing sites
- Product catalogs
```

```typescript
// SSG Example
export async function getStaticProps() {
  const data = await fetchData();
  
  return {
    props: { data },
    // Optional: revalidate every hour
    revalidate: 3600,
  };
}

export default function Page({ data }) {
  return <div>{data.content}</div>;
}
```

### Incremental Static Regeneration (ISR)

```
Flow: Static Shell → Request → Check Age → Regenerate (Background)

Pros:
✓ Fast like SSG
✓ Fresh content
✓ No full rebuilds
✓ Scalable

Cons:
✗ First user sees stale
✗ Complexity
✗ Cache invalidation
✗ Next.js specific

Best For:
- E-commerce products
- News articles
- Content sites
- Frequently updated pages
```

```typescript
// ISR Example
export async function getStaticProps() {
  const data = await fetchData();
  
  return {
    props: { data },
    revalidate: 60, // Revalidate every 60 seconds
  };
}

// On-demand revalidation
export async function POST(req) {
  await res.revalidate('/path');
  return res.json({ revalidated: true });
}
```

### Streaming SSR

```
Flow: Request → Send Shell → Stream Components → Hydrate

Pros:
✓ Fast TTFB
✓ Progressive loading
✓ Better UX
✓ Parallel data fetching

Cons:
✗ Complex implementation
✗ Framework support needed
✗ Debugging challenges
✗ State management

Best For:
- Content-heavy pages
- Dashboards
- Data visualization
- Multi-section pages
```

```typescript
// Streaming SSR Example
export default function Page() {
  return (
    <>
      <Header /> {/* Sent immediately */}
      
      <Suspense fallback={<Loading />}>
        <SlowComponent /> {/* Streams when ready */}
      </Suspense>
    </>
  );
}
```

### Islands Architecture

```
Flow: Static HTML → Island Markers → Hydrate Islands Only

Pros:
✓ Minimal JS
✓ Fast load times
✓ Great performance
✓ Progressive enhancement

Cons:
✗ Island isolation
✗ Limited frameworks
✗ State sharing issues
✗ Learning curve

Best For:
- Content sites
- Marketing pages
- Documentation
- Blogs with widgets
```

```astro
<!-- Islands Example (Astro) -->
<html>
  <body>
    <header>Static Header</header>
    
    <main>
      <p>Static content</p>
      
      <!-- Only this hydrates -->
      <Counter client:visible />
    </main>
    
    <footer>Static Footer</footer>
  </body>
</html>
```

### Progressive Hydration

```
Flow: SSR HTML → Hydrate Critical → Hydrate Secondary → Hydrate Tertiary

Pros:
✓ Fast TTI
✓ Prioritization
✓ Better UX
✓ Flexible

Cons:
✗ Implementation complexity
✗ Testing challenges
✗ State coordination
✗ Framework limitations

Best For:
- Complex pages
- E-commerce
- Dashboards
- Content + interaction mix
```

```typescript
// Progressive Hydration Example
function App() {
  return (
    <>
      <CriticalComponent /> {/* Immediate */}
      
      <LazyHydrate whenIdle>
        <SecondaryComponent />
      </LazyHydrate>
      
      <LazyHydrate whenVisible>
        <TertiaryComponent />
      </LazyHydrate>
    </>
  );
}
```

### Partial Pre-Rendering (PPR)

```
Flow: Static Shell (Build) → Dynamic Holes (Request)

Pros:
✓ Fast TTFB
✓ Fresh dynamic content
✓ Best of both worlds
✓ Great UX

Cons:
✗ Next.js 14+ only
✗ Experimental
✗ Learning curve
✗ Skeleton design needed

Best For:
- E-commerce
- Personalized sites
- Mixed static/dynamic
- Modern applications
```

```typescript
// PPR Example
export default function Page() {
  return (
    <>
      <StaticHeader /> {/* Pre-rendered */}
      
      <Suspense fallback={<Skeleton />}>
        <UserProfile /> {/* Dynamic at runtime */}
      </Suspense>
      
      <StaticContent /> {/* Pre-rendered */}
    </>
  );
}
```

### Edge Rendering

```
Flow: Request → Edge (Near User) → Render → Response

Pros:
✓ Low latency globally
✓ Personalization
✓ Scalable
✓ Fresh content

Cons:
✗ Limited APIs
✗ Memory/CPU limits
✗ Execution time limits
✗ Platform specific

Best For:
- Global apps
- Personalization
- A/B testing
- Geolocation features
```

```typescript
// Edge Rendering Example
export const runtime = 'edge';

export default async function Page() {
  const user = await fetchUserAtEdge();
  
  return (
    <div>
      <h1>Hello, {user.name}!</h1>
      <p>Rendered at edge near you</p>
    </div>
  );
}
```

## Decision Tree

```
Start Here: What type of application?
│
├─ Static Content Site?
│  ├─ Never changes → SSG
│  └─ Changes occasionally → ISR
│
├─ E-commerce Site?
│  ├─ Product pages → PPR or ISR
│  ├─ Checkout → SSR or Edge
│  └─ Admin → CSR
│
├─ News/Media Site?
│  ├─ Articles → SSG + ISR
│  ├─ Live feed → Streaming SSR
│  └─ Comments → Islands or PPR
│
├─ Dashboard/App?
│  ├─ Public dashboard → Streaming SSR
│  ├─ User dashboard → CSR or SSR
│  └─ Admin tools → CSR
│
├─ Global Application?
│  ├─ Personalized → Edge + PPR
│  └─ Static → SSG + CDN
│
└─ Complex Requirements?
   └─ Hybrid: Mix strategies per page!
```

## Performance Characteristics

### Load Time Comparison

```
Typical Page Load Times (Good network, modern device):

CSR:           ████████████████░░░░ 3.5s
SSR:           ████████░░░░░░░░░░░░ 1.5s
SSG:           ██░░░░░░░░░░░░░░░░░░ 0.3s
ISR:           ██░░░░░░░░░░░░░░░░░░ 0.3s
Streaming:     ████░░░░░░░░░░░░░░░░ 0.8s (FCP)
Islands:       ██░░░░░░░░░░░░░░░░░░ 0.4s
Progressive:   ███░░░░░░░░░░░░░░░░░ 0.6s (Critical)
PPR:           ██░░░░░░░░░░░░░░░░░░ 0.3s (Shell)
Edge:          ██░░░░░░░░░░░░░░░░░░ 0.2s

Note: Times vary based on content, complexity, and location
```

### Bundle Size Impact

```
Typical JavaScript Bundle Sizes:

CSR:           ████████████████████ 500KB+
SSR:           ████████████████░░░░ 300KB
SSG:           ████████████████░░░░ 300KB
ISR:           ████████████████░░░░ 300KB
Streaming:     ████████████████░░░░ 300KB
Islands:       ████░░░░░░░░░░░░░░░░ 50KB
Progressive:   ████████████░░░░░░░░ 200KB
PPR:           ████████████░░░░░░░░ 200KB
Edge:          ████████████████░░░░ 300KB
```

## Cost Comparison

### Infrastructure Costs (per 1M requests)

```
CSR:           $5   (Static hosting + client compute)
SSR:           $150 (Server compute + hosting)
SSG:           $2   (CDN only)
ISR:           $30  (CDN + regeneration)
Streaming:     $180 (Server compute premium)
Islands:       $2   (Mostly static)
Progressive:   $120 (Server + optimized compute)
PPR:           $40  (CDN + edge compute)
Edge:          $60  (Edge compute)

Note: Costs vary by provider and usage patterns
```

## Real-World Examples

### E-commerce Product Page

```typescript
// Recommended: PPR
export default function ProductPage({ params }) {
  return (
    <>
      {/* Static: Product info (pre-rendered) */}
      <ProductDetails id={params.id} />
      
      {/* Dynamic: User-specific pricing */}
      <Suspense fallback={<PriceSkeleton />}>
        <UserPricing id={params.id} />
      </Suspense>
      
      {/* Static: Reviews */}
      <Reviews id={params.id} />
      
      {/* Dynamic: Recommendations */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <PersonalizedRecommendations id={params.id} />
      </Suspense>
    </>
  );
}

// Why PPR?
// ✓ Fast TTFB (static shell)
// ✓ Fresh pricing/stock
// ✓ Personalized recommendations
// ✓ SEO-friendly
```

### Blog/News Site

```typescript
// Recommended: SSG + ISR
export async function getStaticProps() {
  const post = await fetchPost();
  
  return {
    props: { post },
    revalidate: 3600, // Revalidate hourly
  };
}

// Why SSG + ISR?
// ✓ Extremely fast delivery
// ✓ Low costs
// ✓ Content updates periodically
// ✓ Scales infinitely
```

### Dashboard Application

```typescript
// Recommended: Streaming SSR or CSR
export default function Dashboard() {
  return (
    <>
      <Header />
      
      {/* Stream data as it loads */}
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />
      </Suspense>
      
      <Suspense fallback={<ChartsSkeleton />}>
        <Charts />
      </Suspense>
      
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
    </>
  );
}

// Why Streaming SSR?
// ✓ Fast FCP (header/layout)
// ✓ Parallel data fetching
// ✓ Progressive loading
// ✓ Better UX than full CSR
```

### Marketing Landing Page

```typescript
// Recommended: Islands Architecture
---
import Counter from './Counter.jsx';
import Newsletter from './Newsletter.vue';
---

<html>
  <body>
    <!-- Static HTML -->
    <header>
      <nav>...</nav>
    </header>
    
    <main>
      <!-- Static content -->
      <section class="hero">
        <h1>Welcome</h1>
      </section>
      
      <!-- Interactive island -->
      <Counter client:visible />
      
      <!-- More static content -->
      <section class="features">...</section>
      
      <!-- Another island -->
      <Newsletter client:idle />
    </main>
    
    <!-- Static footer -->
    <footer>...</footer>
  </body>
</html>

// Why Islands?
// ✓ Minimal JavaScript
// ✓ Fast loading
// ✓ Great Core Web Vitals
// ✓ SEO-friendly
```

## Hybrid Approach

### Combining Strategies

```typescript
// Use different strategies per route
const app = {
  // Static marketing pages
  '/': 'SSG',
  '/about': 'SSG',
  '/pricing': 'SSG',
  
  // Dynamic product pages
  '/products/[id]': 'PPR',
  
  // User dashboard
  '/dashboard': 'Streaming SSR',
  '/dashboard/settings': 'CSR',
  
  // Blog with islands
  '/blog': 'SSG + Islands',
  '/blog/[slug]': 'ISR + Islands',
  
  // API routes at edge
  '/api/*': 'Edge',
};
```

## Migration Paths

### From CSR to Modern Patterns

```
Step 1: Identify static content
  → Move to SSG/SSR for SEO pages

Step 2: Add SSR for dynamic pages
  → Improve SEO and initial load

Step 3: Implement streaming
  → Better UX with progressive loading

Step 4: Add edge rendering
  → Reduce latency globally

Step 5: Optimize with PPR
  → Best of static + dynamic
```

## Common Mistakes

### 1. Using One Pattern Everywhere

```typescript
// BAD: SSR for everything
// Blog posts don't need SSR
export async function getServerSideProps() {
  const post = await fetchPost(); // Unnecessary server hit
  return { props: { post } };
}

// GOOD: SSG for blog posts
export async function getStaticProps() {
  const post = await fetchPost();
  return {
    props: { post },
    revalidate: 3600, // Cache for 1 hour
  };
}
```

### 2. Ignoring Performance Metrics

```typescript
// Monitor what matters for your pattern
const metrics = {
  CSR: ['LCP', 'TTI', 'TBT'],
  SSR: ['TTFB', 'FCP', 'LCP'],
  SSG: ['FCP', 'LCP', 'CLS'],
  Edge: ['TTFB', 'FCP', 'Latency'],
};
```

### 3. Over-Engineering

```typescript
// BAD: Complex setup for simple site
<LazyHydrate whenIdle>
  <ProgressiveComponent>
    <Suspense fallback={<Skeleton />}>
      <SimpleText />
    </Suspense>
  </ProgressiveComponent>
</LazyHydrate>

// GOOD: Keep it simple
<SimpleText />
```

## Best Practices

### 1. Start Simple, Optimize Later

```typescript
// Phase 1: Basic SSR/SSG
// Phase 2: Add ISR for freshness
// Phase 3: Implement streaming
// Phase 4: Add edge rendering
// Phase 5: Optimize with PPR

// Don't jump to Phase 5 immediately!
```

### 2. Measure Performance

```typescript
// Track metrics for your pattern
function trackRendering(strategy: string) {
  const metrics = {
    ttfb: performance.timing.responseStart - performance.timing.requestStart,
    fcp: performance.getEntriesByName('first-contentful-paint')[0]?.startTime,
    lcp: performance.getEntriesByName('largest-contentful-paint')[0]?.startTime,
  };

  analytics.track('rendering_performance', {
    strategy,
    ...metrics,
  });
}
```

### 3. Use Appropriate Caching

```typescript
// Different strategies need different caching
const cacheStrategy = {
  CSR: 'cache client bundles, not data',
  SSR: 'cache at CDN with short TTL',
  SSG: 'cache forever, invalidate on build',
  ISR: 'stale-while-revalidate',
  Edge: 'edge cache with geo-routing',
};
```

## Interview Questions

1. **How do you choose between SSR and SSG?**
   - SSR: Dynamic content, personalization
   - SSG: Static content, performance critical
   - Consider: freshness needs, build time, costs

2. **What is the main benefit of PPR?**
   - Combines SSG speed with SSR flexibility
   - Static shell + dynamic holes
   - Fast TTFB with fresh content

3. **When should you use Edge Rendering?**
   - Global application
   - Low latency critical
   - Personalization at scale
   - Geolocation features

4. **How does Islands Architecture differ from SSR?**
   - Islands: mostly static, minimal JS
   - SSR: full hydration of all components
   - Islands: better performance, limited interactivity

5. **What are trade-offs of Streaming SSR?**
   - Pros: Fast FCP, progressive loading
   - Cons: Complexity, framework support
   - Good for: Multi-section pages

## Key Takeaways

1. No single pattern is best for everything
2. Choose based on content type and requirements
3. Hybrid approaches often work best
4. Measure performance for your use case
5. Start simple, optimize as needed
6. Consider build time, costs, and complexity
7. SSG/ISR for static content
8. SSR/Edge for dynamic/personalized content
9. Islands/PPR for mixed requirements
10. CSR still valid for web applications

## Resources

- [Rendering on the Web](https://web.dev/rendering-on-the-web/)
- [Next.js Rendering](https://nextjs.org/docs/basic-features/pages)
- [Patterns.dev](https://www.patterns.dev/)
- [Web Performance](https://web.dev/performance/)
- [React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023)
