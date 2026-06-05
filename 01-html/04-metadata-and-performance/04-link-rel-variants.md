# Link rel variants

## The Idea

**In plain English:** The `rel` attribute on a `<link>` tag tells the browser what kind of relationship the current page has with another file or resource — for example, whether it is a stylesheet that styles the page, an icon shown in the browser tab, or a translated version of the page in another language.

**Real-world analogy:** Think of a school binder with labeled dividers. Each divider tab tells you what kind of content is behind it — "Homework", "Notes", "Extra Reading". The browser reads `rel` values the same way: each label says what role that linked resource plays.

- The binder = the HTML page
- Each divider tab label = the `rel` attribute value (e.g. `stylesheet`, `icon`, `canonical`)
- The papers behind each divider = the linked file (e.g. the CSS file, the favicon image, the alternate-language URL)

---

## Overview

The `rel` attribute on `<link>` elements defines the relationship between the current document and the linked resource. Different `rel` values serve different purposes for SEO, performance, and functionality.

## Stylesheet links

### Basic stylesheet

```html
<link rel="stylesheet" href="styles.css">
```

### Alternate stylesheet

```html
<link rel="stylesheet" href="default.css" title="Default">
<link rel="alternate stylesheet" href="high-contrast.css" title="High Contrast">
<link rel="alternate stylesheet" href="dark-mode.css" title="Dark Mode">
```

### Media-specific stylesheets

```html
<link rel="stylesheet" href="screen.css" media="screen">
<link rel="stylesheet" href="print.css" media="print">
<link rel="stylesheet" href="mobile.css" media="screen and (max-width: 768px)">
```

## Icon and app links

### Favicon

```html
<link rel="icon" type="image/x-icon" href="/favicon.ico">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="icon" type="image/svg+xml" href="/favicon.svg">
```

### Apple touch icon

```html
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="apple-touch-icon" sizes="152x152" href="/apple-touch-icon-152x152.png">
<link rel="apple-touch-icon" sizes="120x120" href="/apple-touch-icon-120x120.png">
```

### Mask icon (Safari pinned tabs)

```html
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#4F46E5">
```

## Web app manifest

```html
<link rel="manifest" href="/manifest.json">
```

manifest.json:
```json
{
  "name": "My Web App",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#4F46E5",
  "background_color": "#FFFFFF",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

## Canonical link

Indicates the preferred URL for a page:

```html
<link rel="canonical" href="https://example.com/preferred-url/">
```

### Use cases

```html
<!-- Paginated content -->
<link rel="canonical" href="https://example.com/article/">

<!-- URL parameters -->
<!-- example.com/product?color=red&size=large -->
<link rel="canonical" href="https://example.com/product/">

<!-- HTTP vs HTTPS -->
<link rel="canonical" href="https://example.com/page/">

<!-- www vs non-www -->
<link rel="canonical" href="https://example.com/page/">
```

## Alternate links

### Language variants

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
<link rel="alternate" hreflang="de" href="https://example.com/de/">
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

### Regional variants

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/us/">
<link rel="alternate" hreflang="en-GB" href="https://example.com/uk/">
<link rel="alternate" hreflang="en-CA" href="https://example.com/ca/">
<link rel="alternate" hreflang="en-AU" href="https://example.com/au/">
```

### RSS/Atom feeds

```html
<link rel="alternate" type="application/rss+xml" 
      title="RSS Feed" href="/feed.xml">
<link rel="alternate" type="application/atom+xml" 
      title="Atom Feed" href="/atom.xml">
```

### Mobile version

```html
<link rel="alternate" media="only screen and (max-width: 640px)" 
      href="https://m.example.com/page/">
```

### AMP version

```html
<link rel="amphtml" href="https://example.com/page.amp.html">
```

## Resource hints

### DNS prefetch

Resolve domain name early:

```html
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://analytics.google.com">
```

### Preconnect

Establish connection early (DNS + TCP + TLS):

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preconnect" href="https://api.example.com">
```

### Prefetch

Load resource for likely next navigation:

```html
<link rel="prefetch" href="/next-page.html">
<link rel="prefetch" href="/large-image.jpg">
<link rel="prefetch" href="/article-2.html">
```

### Preload

Load critical resource early:

```html
<!-- Fonts -->
<link rel="preload" href="/fonts/main.woff2" as="font" 
      type="font/woff2" crossorigin>

<!-- CSS -->
<link rel="preload" href="/critical.css" as="style">

<!-- JavaScript -->
<link rel="preload" href="/main.js" as="script">

<!-- Images -->
<link rel="preload" href="/hero.jpg" as="image">

<!-- Video -->
<link rel="preload" href="/intro.mp4" as="video">

<!-- Audio -->
<link rel="preload" href="/sound.mp3" as="audio">
```

### Prerender

Fully render page in background:

```html
<link rel="prerender" href="/next-page.html">
```

### Modulepreload

Preload ES modules:

```html
<link rel="modulepreload" href="/app.js">
<link rel="modulepreload" href="/utils.js">
```

## Navigation links

### Previous and next

For paginated content:

```html
<link rel="prev" href="/page-1/">
<link rel="next" href="/page-3/">
```

### First and last

```html
<link rel="first" href="/page-1/">
<link rel="last" href="/page-10/">
```

### Up (parent)

```html
<link rel="up" href="/category/">
```

## Search

OpenSearch description:

```html
<link rel="search" type="application/opensearchdescription+xml" 
      title="Site Search" href="/opensearch.xml">
```

## Author and license

### Author

```html
<link rel="author" href="/humans.txt">
<link rel="author" href="https://example.com/about/">
```

### License

```html
<link rel="license" href="https://creativecommons.org/licenses/by/4.0/">
```

## Pingback and webmention

### Pingback

```html
<link rel="pingback" href="https://example.com/pingback/">
```

### Webmention

```html
<link rel="webmention" href="https://example.com/webmention/">
```

## Help and related

### Help

```html
<link rel="help" href="/help/">
```

### Related

```html
<link rel="related" href="https://example.com/related-article/">
```

## Complete example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <title>Article Title</title>
  
  <!-- Icons -->
  <link rel="icon" type="image/svg+xml" href="/favicon.svg">
  <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
  <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
  <link rel="mask-icon" href="/safari-pinned-tab.svg" color="#4F46E5">
  
  <!-- Web app -->
  <link rel="manifest" href="/manifest.json">
  
  <!-- SEO -->
  <link rel="canonical" href="https://example.com/article/">
  
  <!-- Language variants -->
  <link rel="alternate" hreflang="en" href="https://example.com/en/article/">
  <link rel="alternate" hreflang="es" href="https://example.com/es/article/">
  <link rel="alternate" hreflang="x-default" href="https://example.com/article/">
  
  <!-- Feeds -->
  <link rel="alternate" type="application/rss+xml" href="/feed.xml">
  
  <!-- Pagination -->
  <link rel="prev" href="/article-1/">
  <link rel="next" href="/article-3/">
  
  <!-- Resource hints -->
  <link rel="dns-prefetch" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- Preload critical resources -->
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/hero.jpg" as="image">
  
  <!-- Prefetch next page -->
  <link rel="prefetch" href="/article-3/">
  
  <!-- Stylesheets -->
  <link rel="stylesheet" href="/styles/main.css">
  <link rel="stylesheet" href="/styles/print.css" media="print">
  
  <!-- Scripts -->
  <link rel="modulepreload" href="/js/app.js">
  
  <!-- Other -->
  <link rel="author" href="/about/">
  <link rel="license" href="https://creativecommons.org/licenses/by/4.0/">
</head>
<body>
  <!-- Content -->
</body>
</html>
```

## Resource hint priority

Order from most to least aggressive:

1. **Preload** - Critical resources, loaded immediately
2. **Preconnect** - Early connection for external domains
3. **DNS-prefetch** - DNS lookup for external domains
4. **Prefetch** - Low priority, for likely next navigation
5. **Prerender** - Full page render in background

## Best practices

### DNS prefetch

```html
<!-- Use for external domains you'll need later -->
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://analytics.example.com">
```

### Preconnect

```html
<!-- Use for critical external resources -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- Limit to 4-6 domains (expensive operation) -->
```

### Preload

```html
<!-- Only for critical, above-the-fold resources -->
<link rel="preload" href="/hero.jpg" as="image" fetchpriority="high">
<link rel="preload" href="/critical.css" as="style">
<link rel="preload" href="/main-font.woff2" as="font" type="font/woff2" crossorigin>

<!-- Don't overuse - each preload delays other resources -->
```

### Prefetch

```html
<!-- Use for next page or likely next navigation -->
<link rel="prefetch" href="/next-article.html">
<link rel="prefetch" href="/category-page.html">

<!-- Low priority, doesn't block other resources -->
```

## Common patterns

### Blog post

```html
<link rel="canonical" href="https://blog.example.com/post/">
<link rel="alternate" type="application/rss+xml" href="/feed.xml">
<link rel="prev" href="/previous-post/">
<link rel="next" href="/next-post/">
<link rel="author" href="/about/">
<link rel="license" href="https://creativecommons.org/licenses/by/4.0/">
```

### E-commerce product

```html
<link rel="canonical" href="https://shop.example.com/products/item/">
<link rel="prev" href="/products/prev-item/">
<link rel="next" href="/products/next-item/">
<link rel="preload" href="/product-image-main.jpg" as="image">
<link rel="prefetch" href="/products/related-item/">
```

### Multi-language site

```html
<link rel="canonical" href="https://example.com/en/page/">
<link rel="alternate" hreflang="en" href="https://example.com/en/page/">
<link rel="alternate" hreflang="es" href="https://example.com/es/page/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page/">
<link rel="alternate" hreflang="de" href="https://example.com/de/page/">
<link rel="alternate" hreflang="x-default" href="https://example.com/page/">
```

### News article

```html
<link rel="canonical" href="https://news.example.com/article/">
<link rel="alternate" type="application/rss+xml" href="/feed.xml">
<link rel="amphtml" href="https://news.example.com/article.amp.html">
<link rel="author" href="/journalists/john-doe/">
```

## Performance impact

### Positive impact
- Preconnect to critical domains
- Preload critical resources
- DNS prefetch for external domains
- Prefetch next page

### Negative impact
- Too many preloads (delays other resources)
- Preloading non-critical resources
- Preconnecting to unused domains
- Prerendering pages user won't visit

## Browser support notes

- `dns-prefetch` - Universal support
- `preconnect` - Modern browsers
- `preload` - Modern browsers, critical for fonts
- `prefetch` - Wide support
- `prerender` - Limited support, use sparingly
- `modulepreload` - Modern browsers only

## Key takeaways

- `rel` defines relationship between document and resource
- Use `canonical` to prevent duplicate content issues
- Use `alternate` for language variants and feeds
- Use resource hints to improve performance
- `preconnect` for critical external domains
- `preload` only for critical resources
- `prefetch` for likely next navigation
- Limit preconnect to 4-6 domains
- Always use `crossorigin` for fonts
- Order matters: most critical first
- Test performance impact with DevTools
