# PWA Performance Optimization

## The Idea

**In plain English:** PWA performance optimization is the practice of making a web app load faster and feel smoother, so it responds quickly even on slow internet or older devices. Think of it as trimming all the extra weight off a website so it sprints instead of trudges.

**Real-world analogy:** Imagine a restaurant kitchen during a busy dinner rush. A slow kitchen makes customers wait — but a well-organized one gets food out fast by prepping ingredients ahead of time, only cooking what's ordered, and tracking which dishes take longest.

- The pre-prepped ingredients = cached resources (files stored locally so the browser doesn't re-download them)
- Only cooking what's ordered = code splitting (loading only the JavaScript a user actually needs for the current page)
- Tracking which dishes take longest = performance monitoring (measuring load times so you know what to fix)

---

## Introduction

Performance is crucial for Progressive Web Apps. This guide covers optimization techniques for load times, runtime performance, and user experience through caching strategies, resource optimization, and performance monitoring to ensure your PWA performs like a native app.

## Performance Metrics & Core Web Vitals

```javascript
// Measure Core Web Vitals
function measureWebVitals() {
    // Largest Contentful Paint
    new PerformanceObserver((list) => {
        const entries = list.getEntries();
        const lastEntry = entries[entries.length - 1];
        console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime);
    }).observe({ entryTypes: ['largest-contentful-paint'] });
    
    // First Input Delay
    new PerformanceObserver((list) => {
        list.getEntries().forEach((entry) => {
            console.log('FID:', entry.processingStart - entry.startTime);
        });
    }).observe({ entryTypes: ['first-input'] });
    
    // Cumulative Layout Shift
    let clsScore = 0;
    new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
            if (!entry.hadRecentInput) {
                clsScore += entry.value;
            }
        }
        console.log('CLS:', clsScore);
    }).observe({ entryTypes: ['layout-shift'] });
}

measureWebVitals();
```

## Resource Optimization

### Image Optimization & Lazy Loading

```javascript
class ProgressiveImage {
    constructor(img) {
        this.img = img;
        this.observer = new IntersectionObserver(this.handleIntersection.bind(this));
        this.observer.observe(img);
    }
    
    handleIntersection(entries) {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                this.loadImage();
                this.observer.disconnect();
            }
        });
    }
    
    async loadImage() {
        const lowSrc = this.img.dataset.lowsrc;
        const highSrc = this.img.dataset.src;
        
        if (lowSrc) {
            await this.load(lowSrc, 'loaded-low');
        }
        await this.load(highSrc, 'loaded');
    }
    
    load(src, className) {
        return new Promise((resolve) => {
            const img = new Image();
            img.onload = () => {
                this.img.src = src;
                this.img.classList.add(className);
                resolve();
            };
            img.src = src;
        });
    }
}

document.querySelectorAll('img[data-src]').forEach(img => new ProgressiveImage(img));
```

## Advanced Caching Strategies

```javascript
// sw.js - Smart caching with TTL
class CacheManager {
    constructor(cacheName, maxAge = 24 * 60 * 60 * 1000) {
        this.cacheName = cacheName;
        this.maxAge = maxAge;
    }
    
    async put(request, response) {
        const cache = await caches.open(this.cacheName);
        const headers = new Headers(response.headers);
        headers.append('sw-cache-time', Date.now().toString());
        
        return cache.put(request, new Response(response.body, {
            status: response.status,
            statusText: response.statusText,
            headers
        }));
    }
    
    async match(request) {
        const cache = await caches.open(this.cacheName);
        const response = await cache.match(request);
        
        if (!response) return null;
        
        const cacheTime = response.headers.get('sw-cache-time');
        if (cacheTime && (Date.now() - parseInt(cacheTime)) > this.maxAge) {
            await cache.delete(request);
            return null;
        }
        
        return response;
    }
}
```

## Code Splitting

```javascript
// Dynamic imports for route-based splitting
const routes = {
    '/home': () => import('./pages/home.js'),
    '/about': () => import('./pages/about.js'),
    '/products': () => import('./pages/products.js')
};

async function navigate(path) {
    const loader = routes[path];
    if (loader) {
        const page = await loader();
        page.render();
    }
}
```

## Performance Monitoring

```javascript
class PerformanceMonitor {
    static logMetrics() {
        const perfData = window.performance.timing;
        const pageLoadTime = perfData.loadEventEnd - perfData.navigationStart;
        const connectTime = perfData.responseEnd - perfData.requestStart;
        const renderTime = perfData.domComplete - perfData.domLoading;
        
        console.log({
            pageLoadTime,
            connectTime,
            renderTime,
            domReady: perfData.domContentLoadedEventEnd - perfData.navigationStart
        });
    }
}

window.addEventListener('load', () => PerformanceMonitor.logMetrics());
```

## Browser Support

All modern browsers support PWA performance features.

## Best Practices

1. Measure Core Web Vitals
2. Optimize critical rendering path
3. Implement lazy loading
4. Use code splitting
5. Cache intelligently
6. Monitor continuously
7. Optimize images
8. Minimize JavaScript

## Key Takeaways

- Performance directly impacts user engagement
- Core Web Vitals are critical metrics
- Smart caching improves load times
- Code splitting reduces initial bundle size
- Continuous monitoring is essential

## Resources

- [Web.dev: PWA Performance](https://web.dev/pwa/)
- [Lighthouse](https://developers.google.com/web/tools/lighthouse)
- [Web Vitals](https://web.dev/vitals/)
