# Head essentials

## The Idea

**In plain English:** The `<head>` section of an HTML page is a hidden control panel that gives browsers and search engines important information about the page — things like the page title, what language the text uses, and which files to load — none of which shows up directly on screen.

**Real-world analogy:** Think of a book before you get to chapter one: there's a cover, a copyright page, a table of contents, and maybe an author bio. None of that is the actual story, but it tells you (and librarians) everything they need to know about the book.

- The cover title = the `<title>` tag (shown in browser tabs and search results)
- The copyright page = the `<meta charset>` tag (declares the rules for reading the text)
- The table of contents = `<link>` tags pointing to stylesheets and scripts (tells the browser what extra files to load)

---

## Overview

The `<head>` element contains metadata about the document. It's not displayed but crucial for SEO, performance, accessibility, and functionality.

## Document metadata

### Character encoding

Always declare first in head:

```html
<head>
  <meta charset="UTF-8">
  <!-- Rest of head content -->
</head>
```

### Viewport

Essential for responsive design:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

### Title

Required, used by search engines and browsers:

```html
<title>Page Title | Site Name</title>
```

Best practices:

- Keep under 60 characters
- Put important keywords first
- Include brand name
- Be descriptive and unique per page

## SEO metadata

### Description

```html
<meta name="description" content="Concise description of the page content, 150-160 characters, used in search results.">
```

### Keywords (deprecated but still used)

```html
<meta name="keywords" content="html, metadata, seo, web development">
```

### Author

```html
<meta name="author" content="Your Name">
```

### Robots

```html
<!-- Allow indexing and following links -->
<meta name="robots" content="index, follow">

<!-- Prevent indexing -->
<meta name="robots" content="noindex, nofollow">

<!-- Prevent caching -->
<meta name="robots" content="noarchive">

<!-- Prevent snippet -->
<meta name="robots" content="nosnippet">
```

### Canonical URL

Prevents duplicate content issues:

```html
<link rel="canonical" href="https://example.com/preferred-url/">
```

## Language and localization

### Language declaration

```html
<html lang="en">
  <head>
    <meta charset="UTF-8">
  </head>
</html>
```

### Alternate languages

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

## Favicons and app icons

### Modern favicon set

```html
<!-- SVG favicon (modern browsers) -->
<link rel="icon" type="image/svg+xml" href="/favicon.svg">

<!-- PNG fallback -->
<link rel="icon" type="image/png" href="/favicon-32x32.png" sizes="32x32">
<link rel="icon" type="image/png" href="/favicon-16x16.png" sizes="16x16">

<!-- Apple touch icon -->
<link rel="apple-touch-icon" href="/apple-touch-icon.png" sizes="180x180">

<!-- Legacy ICO -->
<link rel="icon" href="/favicon.ico">
```

### Web app manifest

```html
<link rel="manifest" href="/manifest.json">
```

manifest.json:

```json
{
  "name": "My App",
  "short_name": "App",
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
  ],
  "theme_color": "#4F46E5",
  "background_color": "#FFFFFF",
  "display": "standalone"
}
```

## Theme and appearance

### Theme color

```html
<!-- Browser UI color (mobile Chrome, etc.) -->
<meta name="theme-color" content="#4F46E5">

<!-- With media query for dark mode -->
<meta name="theme-color" content="#4F46E5" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1E1B4B" media="(prefers-color-scheme: dark)">
```

### Apple-specific

```html
<!-- Status bar style -->
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">

<!-- Full screen mode -->
<meta name="apple-mobile-web-app-capable" content="yes">

<!-- App title -->
<meta name="apple-mobile-web-app-title" content="App Name">
```

## Stylesheets

### Basic stylesheet

```html
<link rel="stylesheet" href="styles.css">
```

### Media queries

```html
<link rel="stylesheet" href="print.css" media="print">
<link rel="stylesheet" href="mobile.css" media="screen and (max-width: 768px)">
```

### Alternate stylesheets

```html
<link rel="stylesheet" href="default.css" title="Default">
<link rel="alternate stylesheet" href="high-contrast.css" title="High Contrast">
```

### Preload critical CSS

```html
<link rel="preload" href="critical.css" as="style">
<link rel="stylesheet" href="critical.css">
```

### Inline critical CSS

```html
<style>
  /* Critical above-the-fold CSS */
  body { margin: 0; font-family: system-ui; }
  .hero { min-height: 100vh; }
</style>
```

## Scripts

### Basic script

```html
<script src="script.js"></script>
```

### Defer and async

```html
<!-- Defer: load in order, execute after DOM -->
<script src="main.js" defer></script>

<!-- Async: load and execute ASAP, order not guaranteed -->
<script src="analytics.js" async></script>
```

### Module scripts

```html
<script type="module" src="app.js"></script>
```

### Inline scripts

```html
<script>
  // Inline configuration
  window.APP_CONFIG = {
    apiUrl: 'https://api.example.com'
  };
</script>
```

## Resource hints

### DNS prefetch

```html
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
```

### Preconnect

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### Prefetch

```html
<link rel="prefetch" href="next-page.html">
<link rel="prefetch" href="large-image.jpg">
```

### Preload

```html
<!-- Fonts -->
<link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin>

<!-- Images -->
<link rel="preload" href="hero.jpg" as="image">

<!-- Styles -->
<link rel="preload" href="styles.css" as="style">

<!-- Scripts -->
<link rel="preload" href="script.js" as="script">
```

### Prerender

```html
<link rel="prerender" href="next-page.html">
```

## Complete head example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- Essential meta tags -->
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- Title and description -->
  <title>Page Title - Site Name</title>
  <meta name="description" content="Page description for search engines, 150-160 characters.">
  
  <!-- SEO -->
  <meta name="robots" content="index, follow">
  <link rel="canonical" href="https://example.com/page/">
  
  <!-- Open Graph -->
  <meta property="og:title" content="Page Title">
  <meta property="og:description" content="Description for social sharing">
  <meta property="og:image" content="https://example.com/share-image.jpg">
  <meta property="og:url" content="https://example.com/page/">
  <meta property="og:type" content="website">
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="Page Title">
  <meta name="twitter:description" content="Description for Twitter">
  <meta name="twitter:image" content="https://example.com/share-image.jpg">
  
  <!-- Favicons -->
  <link rel="icon" type="image/svg+xml" href="/favicon.svg">
  <link rel="icon" type="image/png" href="/favicon-32x32.png" sizes="32x32">
  <link rel="apple-touch-icon" href="/apple-touch-icon.png" sizes="180x180">
  
  <!-- Web app manifest -->
  <link rel="manifest" href="/manifest.json">
  
  <!-- Theme color -->
  <meta name="theme-color" content="#4F46E5">
  
  <!-- Resource hints -->
  <link rel="dns-prefetch" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- Preload critical resources -->
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
  
  <!-- Critical CSS inline -->
  <style>
    /* Critical above-the-fold CSS */
    body { margin: 0; font-family: system-ui, -apple-system, sans-serif; }
  </style>
  
  <!-- Stylesheets -->
  <link rel="stylesheet" href="/styles/main.css">
  
  <!-- Scripts -->
  <script src="/js/main.js" defer></script>
  
  <!-- Schema.org structured data -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "WebPage",
    "name": "Page Title",
    "description": "Page description"
  }
  </script>
</head>
<body>
  <!-- Page content -->
</body>
</html>
```

## Performance best practices

### Order matters

1. Charset declaration (first)
2. Viewport meta tag
3. Title
4. Critical resource hints (preconnect, dns-prefetch)
5. Critical preloads
6. Inline critical CSS
7. Stylesheet links
8. Defer/async scripts

### Minimize head size

- Inline only critical CSS
- Use resource hints wisely
- Combine stylesheets where possible
- Defer non-critical scripts
- Minimize number of fonts

### Avoid blocking resources

```html
<!-- Bad: blocks rendering -->
<link rel="stylesheet" href="non-critical.css">
<script src="analytics.js"></script>

<!-- Good: non-blocking -->
<link rel="stylesheet" href="non-critical.css" media="print" 
      onload="this.media='all'">
<script src="analytics.js" async></script>
```

## Security headers

### Content Security Policy

```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self' 'unsafe-inline';">
```

### Referrer Policy

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

## Accessibility

### Skip to content link

```html
<head>
  <!-- Head content -->
</head>
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <!-- Rest of page -->
</body>
```

## Page refresh (use sparingly)

```html
<!-- Redirect after 5 seconds -->
<meta http-equiv="refresh" content="5;url=https://example.com">

<!-- Auto-refresh every 30 seconds (avoid) -->
<meta http-equiv="refresh" content="30">
```

## Base URL

```html
<base href="https://example.com/">
<!-- All relative URLs now resolve against base -->
```

## Best practices checklist

1. Always include charset (UTF-8)
2. Always include viewport meta tag
3. Write unique, descriptive titles
4. Include meta description for all pages
5. Add Open Graph and Twitter Card tags
6. Provide multiple favicon formats
7. Use resource hints for external resources
8. Preload critical resources
9. Inline critical CSS
10. Defer non-critical JavaScript
11. Include structured data (JSON-LD)
12. Set canonical URLs
13. Add language declarations
14. Use semantic ordering in head
15. Test on mobile devices

## Key takeaways

- Head contains metadata, not displayed content
- Charset and viewport are essential
- Title and description critical for SEO
- Use resource hints for performance
- Preload critical resources
- Inline critical CSS
- Defer/async non-critical scripts
- Include favicons and app icons
- Add social media meta tags
- Structure head for optimal loading
