# Resource hints and critical rendering path

## Overview

Resource hints tell browsers which resources to prioritize, and understanding the critical rendering path helps optimize page load performance.

## Critical rendering path

The sequence of steps browsers take to render a page:

1. **Parse HTML** → Build DOM tree
2. **Parse CSS** → Build CSSOM tree
3. **Combine** → Create render tree
4. **Layout** → Calculate positions and sizes
5. **Paint** → Draw pixels to screen

### Blocking resources

#### Render-blocking CSS

```html
<!-- Blocks rendering until loaded -->
<link rel="stylesheet" href="styles.css">
```

#### Parser-blocking JavaScript

```html
<!-- Blocks HTML parsing -->
<script src="script.js"></script>
```

## Resource hints

### dns-prefetch

Resolve DNS early for external domains.

```html
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://analytics.google.com">
```

**When to use:**
- External domains you'll connect to later
- Third-party scripts and fonts
- API endpoints

**Performance impact:**
- Saves 20-120ms per domain
- Very cheap (just DNS lookup)
- Can prefetch many domains

### preconnect

Establish full connection (DNS + TCP + TLS) early.

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preconnect" href="https://api.example.com">
```

**When to use:**
- Critical external resources
- Resources used above the fold
- Known CDN domains

**Performance impact:**
- Saves 100-500ms per connection
- More expensive than dns-prefetch
- Limit to 4-6 connections

**With crossorigin:**
```html
<!-- CORS resources need crossorigin -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### preload

Load critical resources early with high priority.

```html
<!-- Fonts -->
<link rel="preload" href="/fonts/main.woff2" as="font" 
      type="font/woff2" crossorigin>

<!-- Critical CSS -->
<link rel="preload" href="/critical.css" as="style">

<!-- Hero image -->
<link rel="preload" href="/hero.jpg" as="image">

<!-- Critical JavaScript -->
<link rel="preload" href="/main.js" as="script">

<!-- Video -->
<link rel="preload" href="/intro.mp4" as="video" type="video/mp4">
```

**Resource types (as attribute):**
- `font` - Always needs `crossorigin`
- `style` - CSS files
- `script` - JavaScript files
- `image` - Images (jpg, png, webp, etc.)
- `video` - Video files
- `audio` - Audio files
- `document` - HTML documents
- `fetch` - Fetch/XHR requests

**When to use:**
- LCP (Largest Contentful Paint) resources
- Critical fonts
- Above-the-fold images
- Critical scripts/styles

**Performance impact:**
- Significantly improves LCP
- Can delay other resources if overused
- Use sparingly (2-3 critical resources max)

### prefetch

Low-priority loading for likely next navigation.

```html
<!-- Next page -->
<link rel="prefetch" href="/next-page.html">

<!-- Likely next image -->
<link rel="prefetch" href="/product-detail-image.jpg">

<!-- Next article -->
<link rel="prefetch" href="/article-2.html">
```

**When to use:**
- Next page in sequence
- Likely next user action
- Below-the-fold content

**Performance impact:**
- Doesn't block current page
- Loads during browser idle time
- Improves perceived performance

### prerender

Render entire page in background (limited support).

```html
<link rel="prerender" href="/next-page.html">
```

**When to use:**
- High-confidence next navigation
- Search results
- Next step in flow

**Performance impact:**
- Expensive (full page render)
- Use very sparingly
- Limited browser support

### modulepreload

Preload ES modules and dependencies.

```html
<link rel="modulepreload" href="/app.js">
<link rel="modulepreload" href="/components/header.js">
<link rel="modulepreload" href="/utils/helpers.js">
```

**When to use:**
- ES module apps
- Critical module dependencies
- Module bundles

## Critical path optimization strategies

### 1. Inline critical CSS

```html
<head>
  <style>
    /* Critical above-the-fold CSS inline */
    body { margin: 0; font-family: system-ui; }
    .hero { min-height: 100vh; background: blue; }
  </style>
  
  <!-- Non-critical CSS loaded async -->
  <link rel="preload" href="/full.css" as="style" 
        onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/full.css"></noscript>
</head>
```

### 2. Defer non-critical JavaScript

```html
<!-- Critical: inline -->
<script>
  // Critical initialization code
  window.APP_CONFIG = { apiUrl: '/api' };
</script>

<!-- Non-critical: defer -->
<script src="/main.js" defer></script>

<!-- Analytics: async -->
<script src="/analytics.js" async></script>
```

### 3. Optimize font loading

```html
<head>
  <!-- Preconnect to font provider -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- Preload critical fonts -->
  <link rel="preload" href="/fonts/main.woff2" as="font" 
        type="font/woff2" crossorigin>
  
  <!-- Font stylesheet -->
  <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter&display=swap">
</head>
```

### 4. Optimize images

```html
<!-- Preload LCP image -->
<link rel="preload" href="/hero.jpg" as="image" fetchpriority="high">

<!-- Hero image with priority -->
<img src="/hero.jpg" alt="Hero" loading="eager" fetchpriority="high">

<!-- Below-fold images lazy -->
<img src="/content.jpg" alt="Content" loading="lazy">
```

### 5. Reduce render-blocking resources

```html
<!-- Bad: blocks rendering -->
<link rel="stylesheet" href="/full.css">

<!-- Good: critical inline, rest async -->
<style>/* Critical CSS */</style>
<link rel="preload" href="/full.css" as="style" 
      onload="this.rel='stylesheet'">
```

## Complete optimization example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <title>Optimized Page</title>
  
  <!-- DNS prefetch for external domains -->
  <link rel="dns-prefetch" href="https://fonts.googleapis.com">
  <link rel="dns-prefetch" href="https://analytics.google.com">
  
  <!-- Preconnect to critical external resources -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- Preload critical resources -->
  <link rel="preload" href="/fonts/main.woff2" as="font" 
        type="font/woff2" crossorigin>
  <link rel="preload" href="/hero.jpg" as="image" fetchpriority="high">
  <link rel="preload" href="/critical.js" as="script">
  
  <!-- Critical CSS inline -->
  <style>
    body { 
      margin: 0; 
      font-family: system-ui, -apple-system, sans-serif; 
    }
    .hero { 
      min-height: 100vh; 
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      display: flex;
      align-items: center;
      justify-content: center;
      color: white;
    }
  </style>
  
  <!-- Non-critical CSS async -->
  <link rel="preload" href="/styles.css" as="style" 
        onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/styles.css"></noscript>
  
  <!-- Critical JavaScript inline -->
  <script>
    window.APP_CONFIG = {
      apiUrl: '/api',
      version: '1.0.0'
    };
  </script>
  
  <!-- Prefetch likely next page -->
  <link rel="prefetch" href="/about.html">
  <link rel="prefetch" href="/products.html">
</head>
<body>
  <div class="hero">
    <h1>Welcome</h1>
  </div>
  
  <!-- Deferred JavaScript -->
  <script src="/main.js" defer></script>
  <script src="/components.js" defer></script>
  
  <!-- Async third-party scripts -->
  <script src="https://analytics.google.com/analytics.js" async></script>
</body>
</html>
```

## Performance patterns

### Pattern 1: Font optimization

```html
<head>
  <!-- 1. Preconnect -->
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- 2. Preload -->
  <link rel="preload" href="/fonts/main.woff2" as="font" 
        type="font/woff2" crossorigin>
  
  <!-- 3. CSS with font-display -->
  <style>
    @font-face {
      font-family: 'Main';
      src: url('/fonts/main.woff2') format('woff2');
      font-display: swap;
    }
    body { font-family: Main, system-ui, sans-serif; }
  </style>
</head>
```

### Pattern 2: Hero image optimization

```html
<head>
  <!-- Preload hero image -->
  <link rel="preload" href="/hero.jpg" as="image" fetchpriority="high">
</head>
<body>
  <!-- Hero image with priority -->
  <img src="/hero.jpg" alt="Hero" 
       width="1920" height="1080"
       loading="eager" 
       fetchpriority="high">
</body>
```

### Pattern 3: Progressive enhancement

```html
<head>
  <!-- Critical inline -->
  <style>/* Minimal critical CSS */</style>
  
  <!-- Enhanced async -->
  <link rel="preload" href="/enhanced.css" as="style" 
        onload="this.rel='stylesheet'">
</head>
<body>
  <!-- Basic content works without CSS -->
  <main>Content</main>
  
  <!-- Enhanced features deferred -->
  <script src="/enhancements.js" defer></script>
</body>
```

## Measuring performance

### Key metrics

1. **FCP (First Contentful Paint)** - First pixel painted
2. **LCP (Largest Contentful Paint)** - Largest element visible
3. **TTI (Time to Interactive)** - Page becomes interactive
4. **TBT (Total Blocking Time)** - Main thread blocking time
5. **CLS (Cumulative Layout Shift)** - Visual stability

### Chrome DevTools

```javascript
// Performance timing
console.log(performance.getEntriesByType('navigation')[0]);

// Resource timing
console.log(performance.getEntriesByType('resource'));

// Paint timing
console.log(performance.getEntriesByType('paint'));
```

### Web Vitals

```html
<script type="module">
  import {getCLS, getFID, getFCP, getLCP, getTTFB} from 'web-vitals';

  getCLS(console.log);
  getFID(console.log);
  getFCP(console.log);
  getLCP(console.log);
  getTTFB(console.log);
</script>
```

## Best practices

### Do's

1. Preconnect to critical external domains (2-3 max)
2. Preload LCP resources (hero images, fonts)
3. Inline critical CSS (above-the-fold only)
4. Defer non-critical JavaScript
5. Use async for third-party scripts
6. Lazy load below-fold images
7. Use fetchpriority for critical images
8. Minimize render-blocking resources
9. Optimize font loading
10. Measure real-world performance

### Don'ts

1. Don't preload too many resources (delays others)
2. Don't preconnect to unused domains
3. Don't inline all CSS (bloats HTML)
4. Don't use prerender excessively
5. Don't forget crossorigin for fonts
6. Don't block with synchronous scripts
7. Don't load all images eagerly
8. Don't ignore CLS (layout shifts)
9. Don't optimize for one metric only
10. Don't forget mobile performance

## Common mistakes

### Mistake 1: Over-preloading

```html
<!-- Bad: too many preloads -->
<link rel="preload" href="/font1.woff2" as="font">
<link rel="preload" href="/font2.woff2" as="font">
<link rel="preload" href="/font3.woff2" as="font">
<link rel="preload" href="/image1.jpg" as="image">
<link rel="preload" href="/image2.jpg" as="image">
<link rel="preload" href="/image3.jpg" as="image">

<!-- Good: only critical resources -->
<link rel="preload" href="/main-font.woff2" as="font" crossorigin>
<link rel="preload" href="/hero.jpg" as="image">
```

### Mistake 2: Wrong resource hint

```html
<!-- Bad: preload for external resources -->
<link rel="preload" href="https://fonts.googleapis.com/css">

<!-- Good: preconnect for external -->
<link rel="preconnect" href="https://fonts.googleapis.com">
```

### Mistake 3: Missing crossorigin

```html
<!-- Bad: font won't load -->
<link rel="preload" href="/font.woff2" as="font">

<!-- Good: crossorigin required for fonts -->
<link rel="preload" href="/font.woff2" as="font" crossorigin>
```

## Testing tools

1. **Chrome DevTools** - Network, Performance tabs
2. **Lighthouse** - Performance audit
3. **WebPageTest** - Detailed performance analysis
4. **PageSpeed Insights** - Google's recommendations
5. **Chrome UX Report** - Real-world data

## Key takeaways

- Critical rendering path: HTML → CSS → Render → Layout → Paint
- CSS blocks rendering, JavaScript blocks parsing
- Use preconnect for critical external domains (2-3 max)
- Use preload for critical resources (fonts, LCP images)
- Use prefetch for likely next navigation
- Inline critical CSS, async load the rest
- Defer non-critical JavaScript
- Always use crossorigin with preload for fonts
- Limit preloads to 2-3 critical resources
- Measure performance with real-world metrics
- Optimize for LCP, FCP, and CLS
- Test on real devices and networks

---

## Supplementary: Speculation Rules API

The Speculation Rules API is the modern successor to `<link rel="prefetch">` for prerendering full pages. Unlike a prefetch (which only caches the network response), a prerender fully executes the page in a hidden tab so navigation is instant.

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "where": { "href_matches": "/products/*" },
      "eagerness": "moderate"
    }
  ],
  "prefetch": [
    {
      "urls": ["/checkout"],
      "eagerness": "conservative"
    }
  ]
}
</script>
```

### Eagerness Levels

| Level | When it triggers |
|---|---|
| `conservative` | User starts clicking the link |
| `moderate` | User hovers the link for ≥200ms |
| `eager` | Link appears in the viewport |
| `immediate` | As soon as the rule is parsed |

### Programmatic Rules

```javascript
// Add rules dynamically — e.g. after user signals intent
const script = document.createElement('script');
script.type = 'speculationrules';
script.textContent = JSON.stringify({
  prerender: [{ urls: [nextPageUrl] }],
});
document.head.appendChild(script);

// Check if current page was prerendered
if (document.prerendering) {
  // Don't run analytics, start video, etc. until activated
  document.addEventListener('prerenderingchange', () => {
    runAnalytics(); // fires when page is activated (user navigated here)
  });
}
```

### Key Constraints
- Prerendering executes JS — avoid side effects (analytics, payment init) until `prerenderingchange`
- Only works for same-origin pages by default; cross-origin requires `Supports-Loading-Mode: credentialed-prerender` header on the target
- Browser may cancel speculative loads under memory pressure
- Supported: Chrome 108+; Safari/Firefox: not yet
