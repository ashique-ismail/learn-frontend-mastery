# Script attributes

## Overview

The `<script>` element has several attributes that control how and when JavaScript loads and executes. Understanding these attributes is crucial for performance optimization.

## Basic script loading

### Inline script

```html
<script>
  console.log('Inline JavaScript');
  document.body.style.background = 'lightblue';
</script>
```

### External script

```html
<script src="/js/main.js"></script>
```

## Loading strategies

### Default behavior (synchronous)

```html
<script src="/js/script.js"></script>
```

**Behavior:**
- Stops HTML parsing
- Downloads script
- Executes immediately
- Resumes parsing

**Use when:**
- Script must run before page renders
- Script is in `<head>` and needed immediately
- Generally avoid this pattern

### defer attribute

```html
<script src="/js/main.js" defer></script>
```

**Behavior:**
- Downloads in parallel with HTML parsing
- Executes after DOM is ready
- Executes in order (multiple defer scripts)
- Executes before DOMContentLoaded event

**Use when:**
- Script needs full DOM access
- Script order matters
- Non-critical scripts in `<head>`

**Example:**
```html
<head>
  <script src="/js/utils.js" defer></script>
  <script src="/js/main.js" defer></script>
  <script src="/js/init.js" defer></script>
  <!-- Executes in order: utils → main → init -->
</head>
```

### async attribute

```html
<script src="/js/analytics.js" async></script>
```

**Behavior:**
- Downloads in parallel with HTML parsing
- Executes immediately when ready
- Doesn't guarantee order (multiple async scripts)
- May execute before or after DOMContentLoaded

**Use when:**
- Script is independent (analytics, ads)
- Script order doesn't matter
- Script doesn't need DOM

**Example:**
```html
<head>
  <script src="/js/analytics.js" async></script>
  <script src="/js/ads.js" async></script>
  <script src="/js/chat-widget.js" async></script>
  <!-- Executes in random order based on download speed -->
</head>
```

## Comparison table

| Attribute | Parse Blocking | Execution Order | DOM Ready | Use Case |
|-----------|---------------|-----------------|-----------|----------|
| None (default) | Yes | Sequential | Before | Critical inline scripts |
| defer | No | Sequential | After | App scripts needing DOM |
| async | No | Random | Before/After | Independent third-party |

## Visual timeline

```
Normal:  |-HTML-|--[DOWNLOAD]--|-EXECUTE-|-HTML-|
         [BLOCKED PARSING]

Defer:   |-HTML--[DOWNLOAD]--HTML--|DOM|-EXECUTE-|
         [NON-BLOCKING]

Async:   |-HTML--[DOWNLOAD]--[EXEC]-HTML-|
         [EXECUTES WHEN READY]
```

## Module scripts

### ES6 modules

```html
<script type="module" src="/js/app.js"></script>
```

**Characteristics:**
- Deferred by default
- Executed in order
- Strict mode by default
- Can use import/export
- Run once (no duplicate execution)

**Example:**
```html
<!-- app.js -->
<script type="module">
  import { greet } from './utils.js';
  greet('World');
</script>
```

### nomodule fallback

```html
<!-- Modern browsers: use module -->
<script type="module" src="/js/modern.js"></script>

<!-- Legacy browsers: use fallback -->
<script nomodule src="/js/legacy.js"></script>
```

### Module with async

```html
<script type="module" src="/js/app.js" async></script>
```

**Behavior:**
- Downloads in parallel
- Executes when ready (not deferred)
- Still respects module dependencies

## type attribute

### JavaScript (default)

```html
<script src="/js/script.js"></script>
<script type="text/javascript" src="/js/script.js"></script>
```

### Module

```html
<script type="module" src="/js/app.js"></script>
```

### JSON data

```html
<script type="application/json" id="config">
{
  "apiUrl": "https://api.example.com",
  "version": "1.0.0"
}
</script>

<script>
  const config = JSON.parse(
    document.getElementById('config').textContent
  );
  console.log(config.apiUrl);
</script>
```

### Template

```html
<script type="text/template" id="user-template">
  <div class="user">
    <h3>{{name}}</h3>
    <p>{{email}}</p>
  </div>
</script>
```

## crossorigin attribute

For CORS requests and better error reporting.

```html
<!-- No CORS -->
<script src="https://cdn.example.com/library.js"></script>

<!-- With CORS (anonymous) -->
<script src="https://cdn.example.com/library.js" crossorigin></script>
<script src="https://cdn.example.com/library.js" crossorigin="anonymous"></script>

<!-- With CORS (credentials) -->
<script src="https://cdn.example.com/library.js" crossorigin="use-credentials"></script>
```

**Use when:**
- Loading from different origin
- Need detailed error messages
- Using Subresource Integrity (SRI)

## integrity attribute

Subresource Integrity (SRI) for security.

```html
<script 
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous">
</script>
```

**Generates hash:**
```bash
# Generate SRI hash
openssl dgst -sha384 -binary library.js | openssl base64 -A
```

**Use when:**
- Loading from CDN
- Security is critical
- Want to verify file integrity

## referrerpolicy attribute

Controls referrer information sent.

```html
<script src="/js/script.js" referrerpolicy="no-referrer"></script>
<script src="/js/script.js" referrerpolicy="origin"></script>
<script src="/js/script.js" referrerpolicy="strict-origin-when-cross-origin"></script>
```

**Values:**
- `no-referrer` - No referrer sent
- `origin` - Only origin sent
- `strict-origin` - Origin for same-protocol
- `no-referrer-when-downgrade` - No referrer for HTTPS→HTTP

## fetchpriority attribute

Hints at resource priority.

```html
<!-- High priority -->
<script src="/js/critical.js" fetchpriority="high"></script>

<!-- Low priority -->
<script src="/js/analytics.js" fetchpriority="low" async></script>

<!-- Auto (default) -->
<script src="/js/main.js" fetchpriority="auto"></script>
```

## Complete examples

### Performance-optimized page

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Optimized Page</title>
  
  <!-- Critical inline config -->
  <script>
    window.APP_CONFIG = {
      apiUrl: '/api',
      debug: false
    };
  </script>
  
  <!-- Main app scripts (deferred, in order) -->
  <script src="/js/utils.js" defer></script>
  <script src="/js/components.js" defer></script>
  <script src="/js/main.js" defer></script>
  
  <!-- Third-party scripts (async, independent) -->
  <script src="https://analytics.google.com/analytics.js" 
          async 
          crossorigin="anonymous"></script>
  <script src="https://cdn.example.com/widget.js" 
          async></script>
</head>
<body>
  <div id="app"></div>
  
  <!-- Non-critical scripts at end -->
  <script src="/js/feedback-widget.js" defer></script>
</body>
</html>
```

### Module-based app

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Module App</title>
  
  <!-- Main module -->
  <script type="module" src="/js/app.js"></script>
  
  <!-- Legacy fallback -->
  <script nomodule src="/js/legacy-bundle.js"></script>
  
  <!-- Preload module dependencies -->
  <link rel="modulepreload" href="/js/app.js">
  <link rel="modulepreload" href="/js/utils.js">
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

### Secure CDN loading

```html
<script 
  src="https://cdn.jsdelivr.net/npm/vue@3.3.4/dist/vue.global.js"
  integrity="sha384-example-hash-here"
  crossorigin="anonymous"
  defer>
</script>
```

### Analytics pattern

```html
<!-- Google Analytics optimized -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'GA_MEASUREMENT_ID');
</script>
```

## Common patterns

### Pattern 1: Critical + deferred

```html
<head>
  <!-- Critical: inline -->
  <script>
    // Configuration needed immediately
    window.CONFIG = { theme: 'dark' };
  </script>
  
  <!-- Non-critical: deferred -->
  <script src="/js/main.js" defer></script>
</head>
```

### Pattern 2: Multiple deferred (ordered)

```html
<head>
  <!-- Executes in order -->
  <script src="/js/polyfills.js" defer></script>
  <script src="/js/vendor.js" defer></script>
  <script src="/js/app.js" defer></script>
</head>
```

### Pattern 3: Independent async

```html
<head>
  <!-- Executes independently when ready -->
  <script src="/js/analytics.js" async></script>
  <script src="/js/ads.js" async></script>
  <script src="/js/chat.js" async></script>
</head>
```

### Pattern 4: Module with imports

```html
<!-- app.js -->
<script type="module">
  import { init } from './main.js';
  import { trackEvent } from './analytics.js';
  
  init();
  trackEvent('app_loaded');
</script>
```

## Performance best practices

### 1. Defer non-critical scripts

```html
<!-- Bad: blocks rendering -->
<head>
  <script src="/js/main.js"></script>
</head>

<!-- Good: deferred -->
<head>
  <script src="/js/main.js" defer></script>
</head>
```

### 2. Async third-party scripts

```html
<!-- Bad: blocks parsing -->
<script src="https://analytics.com/script.js"></script>

<!-- Good: async -->
<script src="https://analytics.com/script.js" async></script>
```

### 3. Inline critical code

```html
<!-- Good: critical code inline -->
<script>
  // Small, critical initialization
  if (localStorage.theme === 'dark') {
    document.documentElement.classList.add('dark');
  }
</script>
```

### 4. Preload important scripts

```html
<link rel="preload" href="/js/critical.js" as="script">
<script src="/js/critical.js" defer></script>
```

## Script placement

### Head placement

```html
<head>
  <!-- Critical inline -->
  <script>/* Critical code */</script>
  
  <!-- Deferred scripts -->
  <script src="/js/main.js" defer></script>
  
  <!-- Async third-party -->
  <script src="/js/analytics.js" async></script>
</head>
```

### Body end placement (legacy pattern)

```html
<body>
  <!-- Content -->
  
  <!-- Scripts at end (before </body>) -->
  <script src="/js/main.js"></script>
</body>
```

Modern approach uses `defer` in `<head>` instead.

## Error handling

### onerror attribute

```html
<script 
  src="https://cdn.example.com/library.js"
  onerror="console.error('Failed to load script')">
</script>
```

### Global error handler

```html
<script>
  window.addEventListener('error', function(e) {
    if (e.target.tagName === 'SCRIPT') {
      console.error('Script failed:', e.target.src);
    }
  });
</script>
```

## Content Security Policy

### Inline scripts with CSP

```html
<!-- Requires CSP: script-src 'sha256-...' -->
<script>
  console.log('Inline script');
</script>

<!-- Or use nonce -->
<meta http-equiv="Content-Security-Policy" 
      content="script-src 'nonce-random123'">
<script nonce="random123">
  console.log('With nonce');
</script>
```

## Key takeaways

- Default scripts block parsing (avoid in head)
- `defer` downloads in parallel, executes after DOM, preserves order
- `async` downloads in parallel, executes ASAP, random order
- Use `defer` for app scripts that need DOM
- Use `async` for independent third-party scripts
- `type="module"` is deferred by default
- Always use `crossorigin` with `integrity`
- Inline critical scripts in head
- Defer non-critical scripts
- Async third-party scripts
- Module scripts are modern and deferred
- Test script loading with DevTools Network tab
