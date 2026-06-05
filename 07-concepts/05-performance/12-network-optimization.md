# Network Optimization

## The Idea

**In plain English:** Network optimization is about making websites load as fast as possible by controlling how and when files travel between a server (the computer storing your site) and a browser (like Chrome, which displays it). The less waiting the browser has to do, the faster the page appears.

**Real-world analogy:** Imagine ordering food from a restaurant through a delivery driver. You can call ahead to tell the driver which roads to take, pre-order dishes you know you'll want next, and even ask the kitchen to start cooking before you officially place your order — all so the food arrives the moment you're ready for it.

- The delivery driver = the browser fetching files from a server
- Calling ahead to pre-plan the route = using `preconnect` or `dns-prefetch` to prepare the connection early
- Pre-ordering dishes for later = using `prefetch` to download resources you'll need on the next page
- Asking the kitchen to start cooking your current order immediately = using `preload` to fetch critical files for the current page right away

---

## Overview

Network optimization is crucial for fast page loads, especially on slower connections. Modern protocols like HTTP/2 and HTTP/3 enable performance improvements through multiplexing and reduced latency. Resource hints (preload, prefetch, dns-prefetch, preconnect) and priority hints help browsers optimize resource loading. This guide covers HTTP/2 multiplexing, HTTP/3 QUIC protocol, comprehensive resource hint strategies, and modern priority management.

## HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP/1.1 (1997-2015):
┌──────────────────────────────────────────────────────┐
│  Connection 1:  [HTML]─────────────────────────►     │
│  Connection 2:      [CSS]─────────────────────►      │
│  Connection 3:          [JS]──────────────────►      │
│  Connection 4:              [IMG1]────────────►      │
│  Connection 5:                  [IMG2]────────►      │
│  Connection 6:                      [IMG3]────►      │
└──────────────────────────────────────────────────────┘
6-8 connections max per domain, head-of-line blocking

HTTP/2 (2015):
┌──────────────────────────────────────────────────────┐
│  Single Connection (Multiplexed):                    │
│  │ [HTML] [CSS] [JS] [IMG1] [IMG2] [IMG3] │         │
│  └─────────── All parallel ──────────────┘          │
└──────────────────────────────────────────────────────┘
Binary protocol, header compression, server push

HTTP/3 (2020+):
┌──────────────────────────────────────────────────────┐
│  QUIC over UDP (not TCP):                           │
│  │ [Stream1] [Stream2] [Stream3] [Stream4] │        │
│  └──── Independent streams (no HOL blocking) ─┘     │
└──────────────────────────────────────────────────────┘
0-RTT, better mobile performance, built-in encryption
```

## HTTP/2 Multiplexing

HTTP/2 allows multiple requests over a single TCP connection:

### HTTP/2 Features

```
1. Binary Framing:
   ┌──────────────────────────────────────────┐
   │  Frame Type │ Stream ID │ Payload        │
   ├──────────────────────────────────────────┤
   │  HEADERS    │     1     │ :path: /api    │
   │  DATA       │     1     │ {...json...}   │
   │  HEADERS    │     3     │ :path: /style  │
   │  DATA       │     3     │ .class{...}    │
   └──────────────────────────────────────────┘

2. Stream Prioritization:
   ┌─────────────┐
   │   Stream 1  │  (Priority: High - HTML)
   │     ↓       │
   │   Stream 3  │  (Priority: Medium - CSS)
   │     ↓       │
   │   Stream 5  │  (Priority: Low - Image)
   └─────────────┘

3. Header Compression (HPACK):
   Request 1:  :method: GET, :path: /api, cookie: abc123
   Request 2:  :method: GET, :path: /api2  (cookie reused from table)
```

### Detecting HTTP/2

```typescript
// Check if HTTP/2 is supported
async function detectHTTP2Support(url: string): Promise<boolean> {
  try {
    const response = await fetch(url);
    
    // Check protocol via Performance API
    const resources = performance.getEntriesByType('resource') as PerformanceResourceTiming[];
    const resource = resources.find(r => r.name === url);
    
    // nextHopProtocol: 'h2' = HTTP/2, 'h3' = HTTP/3
    const protocol = (resource as any)?.nextHopProtocol || '';
    
    console.log('Protocol:', protocol);
    return protocol.startsWith('h2');
  } catch (error) {
    return false;
  }
}

// Usage
detectHTTP2Support('https://example.com').then(isHTTP2 => {
  console.log('HTTP/2 supported:', isHTTP2);
});
```

### React HTTP/2 Detector

```typescript
// useHTTP2Detection.ts
import { useEffect, useState } from 'react';

export function useHTTP2Detection() {
  const [protocol, setProtocol] = useState<string>('unknown');
  const [isHTTP2, setIsHTTP2] = useState(false);
  const [isHTTP3, setIsHTTP3] = useState(false);
  
  useEffect(() => {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries() as PerformanceResourceTiming[];
      
      for (const entry of entries) {
        const nextHop = (entry as any).nextHopProtocol;
        
        if (nextHop) {
          setProtocol(nextHop);
          setIsHTTP2(nextHop.startsWith('h2'));
          setIsHTTP3(nextHop.startsWith('h3'));
          break;
        }
      }
    });
    
    observer.observe({ entryTypes: ['resource'] });
    
    return () => observer.disconnect();
  }, []);
  
  return { protocol, isHTTP2, isHTTP3 };
}

// Usage
function ProtocolInfo() {
  const { protocol, isHTTP2, isHTTP3 } = useHTTP2Detection();
  
  return (
    <div>
      Protocol: {protocol}
      {isHTTP2 && ' ✓ HTTP/2 enabled'}
      {isHTTP3 && ' ✓ HTTP/3 enabled'}
    </div>
  );
}
```

### HTTP/2 Server Push (deprecated in Chrome)

```typescript
// Server-side (Node.js with HTTP/2)
import http2 from 'http2';
import fs from 'fs';

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  if (headers[':path'] === '/') {
    // Push critical CSS before HTML response
    stream.pushStream({ ':path': '/style.css' }, (err, pushStream) => {
      if (err) throw err;
      
      pushStream.respond({ ':status': 200 });
      pushStream.end(fs.readFileSync('style.css'));
    });
    
    // Send HTML
    stream.respond({
      'content-type': 'text/html',
      ':status': 200
    });
    stream.end(fs.readFileSync('index.html'));
  }
});

// Note: HTTP/2 push is deprecated. Use 103 Early Hints instead
```

## HTTP/3 and QUIC

HTTP/3 uses QUIC protocol over UDP instead of TCP:

```
TCP + TLS + HTTP/2:
┌───────────────────────────────────────────────────┐
│  TCP Handshake (1 RTT)                            │
│  TLS Handshake (1-2 RTT)                          │
│  HTTP Request (1 RTT)                             │
│  Total: 3-4 Round Trips                           │
└───────────────────────────────────────────────────┘

QUIC + HTTP/3:
┌───────────────────────────────────────────────────┐
│  QUIC Handshake (combines TCP + TLS) (1 RTT)     │
│  HTTP Request (0 RTT with 0-RTT resumption)       │
│  Total: 0-1 Round Trips                           │
└───────────────────────────────────────────────────┘
```

### HTTP/3 Benefits

```typescript
/*
1. No Head-of-Line Blocking:
   - TCP: One lost packet blocks all streams
   - QUIC: Streams are independent
   
2. Connection Migration:
   - TCP: Connection tied to IP address
   - QUIC: Connection survives IP changes (WiFi → Cellular)
   
3. 0-RTT Resumption:
   - Previous connection data reused
   - Instant resumption for repeat visitors
   
4. Built-in Encryption:
   - TLS 1.3 integrated into QUIC
   - Always encrypted
*/

// Check if HTTP/3 is available
function checkHTTP3Support(): boolean {
  const entries = performance.getEntriesByType('resource') as PerformanceResourceTiming[];
  
  return entries.some(entry => 
    (entry as any).nextHopProtocol?.startsWith('h3')
  );
}

// Alternative Service header detection (server sends)
// Alt-Svc: h3=":443"; ma=86400
```

## Resource Hints

Resource hints tell the browser to optimize resource loading:

### 1. dns-prefetch

```html
<!-- Resolve DNS early for external domains -->
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://analytics.example.com">

<!-- Good for: External resources that will be loaded later -->
```

```typescript
// React dynamic dns-prefetch
function DnsPrefetch({ hosts }: { hosts: string[] }) {
  return (
    <>
      {hosts.map(host => (
        <link key={host} rel="dns-prefetch" href={host} />
      ))}
    </>
  );
}

// Usage
const externalHosts = [
  'https://cdn.example.com',
  'https://api.example.com'
];

<DnsPrefetch hosts={externalHosts} />
```

### 2. preconnect

```html
<!-- DNS + TCP + TLS handshake -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- 
  Use for: Critical external resources
  Cost: More expensive than dns-prefetch
  Limit: Only 2-3 preconnect hints (browser limit)
-->
```

```typescript
// React Preconnect Component
interface PreconnectProps {
  urls: Array<{
    href: string;
    crossorigin?: boolean;
  }>;
}

function Preconnect({ urls }: PreconnectProps) {
  return (
    <>
      {urls.map((url, index) => (
        <link
          key={index}
          rel="preconnect"
          href={url.href}
          {...(url.crossorigin && { crossOrigin: 'anonymous' })}
        />
      ))}
    </>
  );
}

// Usage
<Preconnect 
  urls={[
    { href: 'https://fonts.googleapis.com' },
    { href: 'https://fonts.gstatic.com', crossorigin: true }
  ]}
/>
```

### 3. prefetch

```html
<!-- Download resource for FUTURE navigation -->
<link rel="prefetch" href="/next-page.html" as="document">
<link rel="prefetch" href="/next-page.js" as="script">
<link rel="prefetch" href="/images/hero-next.jpg" as="image">

<!-- 
  Use for: Likely next navigation
  Priority: Lowest priority (idle time only)
  Cache: Cached for 5 minutes
-->
```

```typescript
// React Prefetch Hook
import { useEffect } from 'react';

export function usePrefetch(urls: string[]) {
  useEffect(() => {
    const links: HTMLLinkElement[] = [];
    
    urls.forEach(url => {
      const link = document.createElement('link');
      link.rel = 'prefetch';
      link.href = url;
      link.as = getAsType(url);
      
      document.head.appendChild(link);
      links.push(link);
    });
    
    return () => {
      links.forEach(link => {
        document.head.removeChild(link);
      });
    };
  }, [urls]);
}

function getAsType(url: string): string {
  if (url.endsWith('.js')) return 'script';
  if (url.endsWith('.css')) return 'style';
  if (url.match(/\.(jpg|png|webp)$/)) return 'image';
  return 'fetch';
}

// Usage
function HomePage() {
  // Prefetch likely next pages
  usePrefetch([
    '/about',
    '/products',
    '/products/hero.jpg'
  ]);
  
  return <div>Home Page</div>;
}
```

### 4. preload

```html
<!-- Download resource for CURRENT page (high priority) -->
<link rel="preload" href="/critical.css" as="style">
<link rel="preload" href="/hero.jpg" as="image">
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin>

<!-- 
  Use for: Critical resources discovered late
  Priority: High priority
  Warning: Don't overuse (blocks other resources)
-->
```

```typescript
// React Preload Component
interface PreloadProps {
  resources: Array<{
    href: string;
    as: string;
    type?: string;
    crossorigin?: boolean;
  }>;
}

function Preload({ resources }: PreloadProps) {
  return (
    <>
      {resources.map((resource, index) => (
        <link
          key={index}
          rel="preload"
          href={resource.href}
          as={resource.as}
          {...(resource.type && { type: resource.type })}
          {...(resource.crossorigin && { crossOrigin: 'anonymous' })}
        />
      ))}
    </>
  );
}

// Usage
<Preload 
  resources={[
    { href: '/fonts/inter.woff2', as: 'font', type: 'font/woff2', crossorigin: true },
    { href: '/hero.jpg', as: 'image' },
    { href: '/critical.css', as: 'style' }
  ]}
/>
```

### 5. modulepreload

```html
<!-- Preload ES modules with dependencies -->
<link rel="modulepreload" href="/app.js">
<link rel="modulepreload" href="/utils.js">

<!-- Also preloads module dependencies automatically -->
```

```typescript
// React Module Preload
function ModulePreload({ modules }: { modules: string[] }) {
  return (
    <>
      {modules.map(module => (
        <link key={module} rel="modulepreload" href={module} />
      ))}
    </>
  );
}

// Usage
<ModulePreload modules={['/main.js', '/vendor.js']} />
```

### Angular Resource Hints Service

```typescript
// resource-hints.service.ts
import { Injectable, Inject, PLATFORM_ID } from '@angular/core';
import { DOCUMENT, isPlatformBrowser } from '@angular/common';

interface ResourceHint {
  rel: 'dns-prefetch' | 'preconnect' | 'prefetch' | 'preload' | 'modulepreload';
  href: string;
  as?: string;
  type?: string;
  crossorigin?: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class ResourceHintsService {
  private addedHints = new Set<string>();
  
  constructor(
    @Inject(DOCUMENT) private document: Document,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}
  
  addHint(hint: ResourceHint): void {
    if (!isPlatformBrowser(this.platformId)) return;
    
    const key = `${hint.rel}:${hint.href}`;
    if (this.addedHints.has(key)) return;
    
    const link = this.document.createElement('link');
    link.rel = hint.rel;
    link.href = hint.href;
    
    if (hint.as) link.setAttribute('as', hint.as);
    if (hint.type) link.type = hint.type;
    if (hint.crossorigin) link.crossOrigin = 'anonymous';
    
    this.document.head.appendChild(link);
    this.addedHints.add(key);
  }
  
  dnsPrefetch(hosts: string[]): void {
    hosts.forEach(href => {
      this.addHint({ rel: 'dns-prefetch', href });
    });
  }
  
  preconnect(urls: Array<{ href: string; crossorigin?: boolean }>): void {
    urls.forEach(url => {
      this.addHint({ 
        rel: 'preconnect', 
        href: url.href,
        crossorigin: url.crossorigin
      });
    });
  }
  
  prefetch(resources: Array<{ href: string; as: string }>): void {
    resources.forEach(resource => {
      this.addHint({
        rel: 'prefetch',
        href: resource.href,
        as: resource.as
      });
    });
  }
  
  preload(resources: Array<ResourceHint>): void {
    resources.forEach(resource => {
      this.addHint({ ...resource, rel: 'preload' });
    });
  }
}

// Usage in component
@Component({
  selector: 'app-home',
  template: '<div>Home</div>'
})
export class HomeComponent implements OnInit {
  constructor(private hints: ResourceHintsService) {}
  
  ngOnInit(): void {
    // Preconnect to critical origins
    this.hints.preconnect([
      { href: 'https://api.example.com' },
      { href: 'https://cdn.example.com', crossorigin: true }
    ]);
    
    // Preload critical resources
    this.hints.preload([
      { href: '/hero.jpg', as: 'image' },
      { href: '/critical.css', as: 'style' }
    ]);
    
    // Prefetch next likely page
    this.hints.prefetch([
      { href: '/about', as: 'document' }
    ]);
  }
}
```

## Priority Hints

Control resource loading priority with `fetchpriority` attribute:

```html
<!-- Boost priority of hero image -->
<img src="/hero.jpg" fetchpriority="high" alt="Hero">

<!-- Lower priority for below-fold images -->
<img src="/secondary.jpg" fetchpriority="low" alt="Secondary">

<!-- Default priority -->
<img src="/normal.jpg" fetchpriority="auto" alt="Normal">

<!-- Priority for scripts -->
<script src="/critical.js" fetchpriority="high"></script>
<script src="/analytics.js" fetchpriority="low"></script>

<!-- Priority for fetch requests -->
<link rel="preload" href="/critical.css" as="style" fetchpriority="high">
```

```typescript
// React Priority Image Component
interface PriorityImageProps {
  src: string;
  alt: string;
  priority: 'high' | 'low' | 'auto';
  loading?: 'lazy' | 'eager';
}

function PriorityImage({ 
  src, 
  alt, 
  priority, 
  loading = 'lazy' 
}: PriorityImageProps) {
  return (
    <img
      src={src}
      alt={alt}
      fetchpriority={priority}
      loading={loading}
    />
  );
}

// Usage
function Hero() {
  return (
    <>
      {/* Above-the-fold hero: high priority, eager loading */}
      <PriorityImage 
        src="/hero.jpg" 
        alt="Hero" 
        priority="high" 
        loading="eager"
      />
      
      {/* Below-the-fold: low priority, lazy loading */}
      <PriorityImage 
        src="/secondary.jpg" 
        alt="Secondary" 
        priority="low" 
        loading="lazy"
      />
    </>
  );
}
```

### Fetch with Priority

```typescript
// High priority fetch
fetch('/critical-data', {
  // @ts-ignore - fetchpriority not in TS types yet
  priority: 'high'
});

// Low priority fetch
fetch('/analytics-data', {
  // @ts-ignore
  priority: 'low'
});

// React Hook with Priority
function usePriorityFetch<T>(
  url: string,
  priority: 'high' | 'low' | 'auto' = 'auto'
) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url, {
          // @ts-ignore
          priority
        });
        
        const json = await response.json();
        setData(json);
      } catch (err) {
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchData();
  }, [url, priority]);
  
  return { data, loading, error };
}

// Usage
function CriticalData() {
  const { data, loading } = usePriorityFetch('/critical', 'high');
  
  if (loading) return <div>Loading...</div>;
  return <div>{JSON.stringify(data)}</div>;
}
```

## Resource Hint Decision Matrix

```
┌──────────────────┬─────────────────┬──────────────────┬─────────────┐
│ Hint             │ When            │ Priority         │ Use Case    │
├──────────────────┼─────────────────┼──────────────────┼─────────────┤
│ dns-prefetch     │ Future          │ Lowest (DNS)     │ External    │
│                  │                 │                  │ domains     │
├──────────────────┼─────────────────┼──────────────────┼─────────────┤
│ preconnect       │ Very soon       │ Low (DNS+TCP+    │ Critical    │
│                  │                 │ TLS)             │ origins     │
├──────────────────┼─────────────────┼──────────────────┼─────────────┤
│ prefetch         │ Next navigation │ Lowest (idle)    │ Next pages  │
├──────────────────┼─────────────────┼──────────────────┼─────────────┤
│ preload          │ Current page    │ High             │ Critical    │
│                  │                 │                  │ resources   │
├──────────────────┼─────────────────┼──────────────────┼─────────────┤
│ modulepreload    │ Current page    │ High             │ ES modules  │
└──────────────────┴─────────────────┴──────────────────┴─────────────┘
```

## Advanced: 103 Early Hints

Server sends hints before the main response:

```typescript
// Server-side (Node.js)
import { createServer } from 'http';

createServer((req, res) => {
  if (req.url === '/') {
    // Send 103 Early Hints
    res.writeHead(103, {
      'Link': [
        '</style.css>; rel=preload; as=style',
        '</script.js>; rel=preload; as=script',
        '</font.woff2>; rel=preload; as=font; crossorigin'
      ].join(', ')
    });
    
    // Later send actual response
    setTimeout(() => {
      res.writeHead(200, { 'Content-Type': 'text/html' });
      res.end('<html>...</html>');
    }, 100);
  }
}).listen(3000);

// Browser receives hints early and starts downloading
// before HTML arrives!
```

## Common Mistakes

### 1. Too Many preconnect Hints

```html
<!-- ❌ Bad: Too many preconnects -->
<link rel="preconnect" href="https://cdn1.example.com">
<link rel="preconnect" href="https://cdn2.example.com">
<link rel="preconnect" href="https://cdn3.example.com">
<link rel="preconnect" href="https://cdn4.example.com">
<link rel="preconnect" href="https://cdn5.example.com">
<!-- Wastes connections, delays critical resources -->

<!-- ✅ Good: Limit to 2-3 critical origins -->
<link rel="preconnect" href="https://cdn.example.com">
<link rel="preconnect" href="https://api.example.com">
```

### 2. Preloading Too Many Resources

```html
<!-- ❌ Bad: Preloading everything -->
<link rel="preload" href="/all-images.jpg" as="image">
<link rel="preload" href="/every-script.js" as="script">
<link rel="preload" href="/all-fonts.woff2" as="font">
<!-- Blocks bandwidth for critical resources -->

<!-- ✅ Good: Only critical, above-the-fold resources -->
<link rel="preload" href="/hero.jpg" as="image">
<link rel="preload" href="/critical.woff2" as="font" crossorigin>
```

### 3. Missing crossorigin on Font Preload

```html
<!-- ❌ Bad: Missing crossorigin (double download) -->
<link rel="preload" href="/font.woff2" as="font">

<!-- ✅ Good: Include crossorigin -->
<link rel="preload" href="/font.woff2" as="font" crossorigin="anonymous">
```

### 4. Wrong Resource Hint for Use Case

```html
<!-- ❌ Bad: Prefetch for current page -->
<link rel="prefetch" href="/critical.css" as="style">
<!-- Will load with lowest priority! -->

<!-- ✅ Good: Preload for current page -->
<link rel="preload" href="/critical.css" as="style">
```

### 5. Not Using HTTP/2 Properly

```typescript
// ❌ Bad: Domain sharding with HTTP/2
// Don't split resources across multiple domains
// HTTP/2 multiplexing works best with single origin

// ✅ Good: Single domain with HTTP/2
// All resources from same origin benefit from multiplexing
```

## Best Practices

1. **Use HTTP/2 or HTTP/3** - enable on server for multiplexing benefits
2. **Limit preconnect to 2-3 origins** - only for critical external domains
3. **Preload only critical resources** - fonts, hero images, critical CSS
4. **Use prefetch for next navigation** - likely next pages/resources
5. **Add crossorigin to font preloads** - avoid double downloads
6. **Use priority hints** - fetchpriority for important vs secondary resources
7. **dns-prefetch for analytics/ads** - domains that will be used later
8. **Monitor with Resource Timing API** - verify hints are working
9. **Don't preload too much** - blocks bandwidth for other resources
10. **Use Early Hints (103)** - send preload hints before HTML response

## When to Use Each Hint

### dns-prefetch
- External analytics domains
- Social media widgets
- Ads networks
- Any external domain you'll use soon

### preconnect
- Critical API servers
- CDN domains with critical resources
- Font providers
- Limit to 2-3 per page

### prefetch
- Next likely navigation pages
- Images for next page
- Data for next view
- Only when user intent is clear

### preload
- Critical fonts (above-fold)
- Hero images
- Critical CSS
- Scripts needed immediately
- Limit to 3-5 per page

### modulepreload
- Critical ES modules
- Module dependencies
- Modern bundler outputs

## Interview Questions

### 1. What is the difference between preload and prefetch?

**Answer**: `preload` loads resources for the CURRENT page with HIGH priority, telling the browser "you will need this soon." `prefetch` loads resources for FUTURE navigations with LOWEST priority during idle time. Preload blocks bandwidth for critical resources, while prefetch uses leftover bandwidth. Use preload for critical current-page resources (fonts, hero images), prefetch for next-page resources.

### 2. Explain HTTP/2 multiplexing and its benefits.

**Answer**: HTTP/2 multiplexing allows multiple requests/responses over a single TCP connection simultaneously, unlike HTTP/1.1 which requires multiple connections (limited to 6-8). Benefits: eliminates connection overhead, removes domain sharding need, better compression, stream prioritization. However, TCP head-of-line blocking still exists - one lost packet blocks all streams. HTTP/3 (QUIC) fixes this with independent streams over UDP.

### 3. Why is the crossorigin attribute required for font preloading?

**Answer**: Fonts are always fetched in CORS mode (crossorigin), even from the same origin. Without `crossorigin` on the preload link, the browser fetches the font twice: once for preload (non-CORS) and again for @font-face (CORS mode), because they're treated as different resources. Adding `crossorigin="anonymous"` ensures the preloaded font matches the actual CORS mode.

### 4. What are the advantages of HTTP/3 over HTTP/2?

**Answer**: HTTP/3 (QUIC) advantages: 
1. No head-of-line blocking (independent streams)
2. Faster connection (1 RTT vs 3-4 RTT)
3. 0-RTT resumption for repeat visitors
4. Connection migration (survives IP changes)
5. Built-in encryption (TLS 1.3)
6. Better for mobile/unreliable networks
HTTP/3 runs over UDP instead of TCP, eliminating TCP-level HOL blocking.

### 5. When should you use dns-prefetch vs preconnect?

**Answer**: Use `dns-prefetch` for domains you'll use later or aren't critical (analytics, ads, social widgets). It only resolves DNS (cheapest). Use `preconnect` for critical external origins you'll use very soon (API servers, CDNs with critical resources) as it performs DNS + TCP + TLS handshake (expensive but ready to use). Limit preconnect to 2-3 per page; use dns-prefetch for others.

### 6. What is priority hints (fetchpriority) and when should you use it?

**Answer**: `fetchpriority` attribute controls resource loading priority: `high` (boost priority), `low` (lower priority), `auto` (browser default). Use on images, scripts, link tags, and fetch() requests. Use cases: `high` for hero images/critical scripts, `low` for below-fold images/analytics, `auto` for normal content. Helps browser make better priority decisions when it can't infer importance from HTML position.

### 7. How would you optimize resource loading for a single-page application?

**Answer**:
1. Preconnect to API/CDN origins
2. Preload critical route resources
3. Prefetch likely next route bundles
4. Use HTTP/2/3 for multiplexing
5. Code split by route
6. Lazy load below-fold components
7. Priority hints for hero images
8. Module preload for critical chunks
9. Service worker for offline/cache
10. Monitor with Resource Timing API

### 8. What is the Resource Timing API and how do you use it to verify resource hints?

**Answer**: Resource Timing API provides performance metrics for each resource loaded (duration, size, timing breakdown). Access via `performance.getEntriesByType('resource')`. Verify hints by checking: `connectStart - fetchStart` (should be ~0 for preconnect), `duration` (should be faster with preload), `transferSize` (cache validation), `nextHopProtocol` (HTTP/2 vs HTTP/3). Compare preloaded vs non-preloaded resources to measure hint effectiveness.

## Key Takeaways

1. **HTTP/2 multiplexing eliminates need for domain sharding** - use single origin
2. **HTTP/3 (QUIC) provides faster connections** - especially on mobile/poor networks
3. **Preload for current page critical resources** - fonts, hero images, critical CSS
4. **Prefetch for next navigation resources** - likely next pages during idle time
5. **Limit preconnect to 2-3 origins** - expensive but worth it for critical domains
6. **dns-prefetch for non-critical external domains** - analytics, ads, social
7. **Always add crossorigin to font preloads** - avoid double downloads
8. **Use priority hints (fetchpriority)** - boost/lower resource priority
9. **Monitor with Resource Timing API** - verify optimizations are working
10. **103 Early Hints for fastest loading** - send preload hints before HTML response

## Resources

- [HTTP/2 Explained (web.dev)](https://web.dev/performance-http2/)
- [HTTP/3 (Cloudflare)](https://www.cloudflare.com/learning/performance/what-is-http3/)
- [Resource Hints (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTML/Link_types)
- [Priority Hints (web.dev)](https://web.dev/priority-hints/)
- [Resource Timing API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Resource_Timing_API)
- [103 Early Hints (Chrome Developers)](https://developer.chrome.com/blog/early-hints/)
- [Preload, Prefetch and Priorities (MDN)](https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work)
