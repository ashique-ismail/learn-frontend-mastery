# Core Web Vitals

## Overview

Core Web Vitals are a set of specific metrics that Google considers essential for measuring user experience on the web. These metrics focus on three aspects of user experience: loading performance, interactivity, and visual stability. Understanding and optimizing for Core Web Vitals is crucial for SEO, user satisfaction, and overall web performance.

## The Three Core Web Vitals

### 1. Largest Contentful Paint (LCP)

LCP measures loading performance by tracking when the largest content element becomes visible in the viewport.

**Good LCP**: ≤ 2.5 seconds
**Needs Improvement**: 2.5-4.0 seconds
**Poor LCP**: > 4.0 seconds

```typescript
// Measuring LCP using Web Vitals library
import { onLCP } from 'web-vitals';

onLCP((metric) => {
  console.log('LCP:', metric.value);
  
  // Send to analytics
  sendToAnalytics({
    name: 'LCP',
    value: metric.value,
    id: metric.id,
    delta: metric.delta,
    rating: metric.rating // 'good', 'needs-improvement', or 'poor'
  });
});

// React hook for LCP monitoring
import { useEffect } from 'react';

function useLCP(callback: (metric: any) => void) {
  useEffect(() => {
    const unsubscribe = onLCP(callback);
    return () => {
      // Cleanup if needed
    };
  }, [callback]);
}

// Usage in component
function App() {
  useLCP((metric) => {
    console.log('LCP detected:', metric);
  });
  
  return <div>Your app content</div>;
}
```

**Angular Implementation:**

```typescript
// lcp-monitor.service.ts
import { Injectable } from '@angular/core';
import { onLCP } from 'web-vitals';

@Injectable({
  providedIn: 'root'
})
export class LCPMonitorService {
  private lcpValue: number | null = null;

  startMonitoring(): void {
    onLCP((metric) => {
      this.lcpValue = metric.value;
      
      console.log('LCP:', {
        value: metric.value,
        rating: metric.rating,
        element: metric.entries[metric.entries.length - 1]?.element
      });

      this.sendToAnalytics(metric);
    });
  }

  private sendToAnalytics(metric: any): void {
    // Send to your analytics service
    if (typeof window !== 'undefined' && (window as any).gtag) {
      (window as any).gtag('event', 'web_vitals', {
        event_category: 'Web Vitals',
        event_label: 'LCP',
        value: Math.round(metric.value),
        metric_id: metric.id,
        metric_rating: metric.rating
      });
    }
  }

  getLCPValue(): number | null {
    return this.lcpValue;
  }
}

// app.component.ts
import { Component, OnInit } from '@angular/core';
import { LCPMonitorService } from './lcp-monitor.service';

@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent implements OnInit {
  constructor(private lcpMonitor: LCPMonitorService) {}

  ngOnInit(): void {
    this.lcpMonitor.startMonitoring();
  }
}
```

### 2. First Input Delay (FID) / Interaction to Next Paint (INP)

**FID** measures the time from when a user first interacts with your page to when the browser can respond. As of March 2024, **INP** replaces FID as a Core Web Vital.

**INP** measures the latency of all user interactions throughout the page lifecycle.

**Good INP**: ≤ 200ms
**Needs Improvement**: 200-500ms
**Poor INP**: > 500ms

```typescript
// Measuring INP
import { onINP } from 'web-vitals';

onINP((metric) => {
  console.log('INP:', metric.value);
  
  sendToAnalytics({
    name: 'INP',
    value: metric.value,
    rating: metric.rating
  });
});

// React: Optimizing for better INP
import { useCallback, useTransition } from 'react';

function SearchComponent() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleSearch = useCallback((value: string) => {
    setQuery(value);
    
    // Mark expensive operations as transitions
    startTransition(() => {
      const filtered = expensiveFilterOperation(value);
      setResults(filtered);
    });
  }, []);

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      {isPending && <div>Updating...</div>}
      <ResultsList results={results} />
    </div>
  );
}

// Debouncing to reduce interaction frequency
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function OptimizedSearch() {
  const [input, setInput] = useState('');
  const debouncedInput = useDebounce(input, 300);

  useEffect(() => {
    if (debouncedInput) {
      performSearch(debouncedInput);
    }
  }, [debouncedInput]);

  return (
    <input
      value={input}
      onChange={(e) => setInput(e.target.value)}
    />
  );
}
```

**Angular Implementation:**

```typescript
// inp-monitor.service.ts
import { Injectable } from '@angular/core';
import { onINP } from 'web-vitals';

@Injectable({
  providedIn: 'root'
})
export class INPMonitorService {
  startMonitoring(): void {
    onINP((metric) => {
      console.log('INP:', metric);
      
      // Identify problematic interactions
      if (metric.rating === 'poor') {
        console.warn('Poor INP detected:', {
          value: metric.value,
          entries: metric.entries
        });
      }
    });
  }
}

// Debounce directive for better INP
import { Directive, EventEmitter, HostListener, Input, Output } from '@angular/core';
import { Subject } from 'rxjs';
import { debounceTime } from 'rxjs/operators';

@Directive({
  selector: '[appDebounce]'
})
export class DebounceDirective {
  @Input() debounceTime = 300;
  @Output() debounceEvent = new EventEmitter();
  private subject = new Subject();

  constructor() {
    this.subject
      .pipe(debounceTime(this.debounceTime))
      .subscribe(value => this.debounceEvent.emit(value));
  }

  @HostListener('input', ['$event'])
  onInput(event: any): void {
    this.subject.next(event);
  }
}

// Usage
@Component({
  selector: 'app-search',
  template: `
    <input
      type="text"
      appDebounce
      [debounceTime]="300"
      (debounceEvent)="onSearch($event)"
      placeholder="Search..."
    />
  `
})
export class SearchComponent {
  onSearch(event: any): void {
    const query = event.target.value;
    console.log('Search query:', query);
  }
}
```

### 3. Cumulative Layout Shift (CLS)

CLS measures visual stability by quantifying unexpected layout shifts during the page's lifecycle.

**Good CLS**: ≤ 0.1
**Needs Improvement**: 0.1-0.25
**Poor CLS**: > 0.25

```typescript
// Measuring CLS
import { onCLS } from 'web-vitals';

onCLS((metric) => {
  console.log('CLS:', metric.value);
  
  // Log layout shift entries for debugging
  metric.entries.forEach((entry) => {
    console.log('Layout shift:', {
      value: entry.value,
      sources: entry.sources,
      hadRecentInput: entry.hadRecentInput
    });
  });
  
  sendToAnalytics({
    name: 'CLS',
    value: metric.value,
    rating: metric.rating
  });
});

// React: Preventing layout shifts with image dimensions
function OptimizedImage({ src, alt }: { src: string; alt: string }) {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  useEffect(() => {
    const img = new Image();
    img.src = src;
    img.onload = () => {
      setDimensions({
        width: img.naturalWidth,
        height: img.naturalHeight
      });
    };
  }, [src]);

  return (
    <img
      src={src}
      alt={alt}
      width={dimensions.width}
      height={dimensions.height}
      style={{ maxWidth: '100%', height: 'auto' }}
    />
  );
}

// Better approach: Always specify dimensions
function ProperImage() {
  return (
    <img
      src="/image.jpg"
      alt="Description"
      width={800}
      height={600}
      style={{ maxWidth: '100%', height: 'auto' }}
    />
  );
}

// Using aspect ratio to prevent CLS
function AspectRatioImage() {
  return (
    <div
      style={{
        position: 'relative',
        aspectRatio: '16 / 9', // Modern CSS
        width: '100%'
      }}
    >
      <img
        src="/image.jpg"
        alt="Description"
        style={{
          position: 'absolute',
          width: '100%',
          height: '100%',
          objectFit: 'cover'
        }}
      />
    </div>
  );
}

// Preventing CLS with skeleton screens
function ContentWithSkeleton() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchData().then(result => {
      setData(result);
      setLoading(false);
    });
  }, []);

  if (loading) {
    return (
      <div className="skeleton">
        <div className="skeleton-line" style={{ width: '100%', height: 20 }} />
        <div className="skeleton-line" style={{ width: '80%', height: 20 }} />
        <div className="skeleton-line" style={{ width: '90%', height: 20 }} />
      </div>
    );
  }

  return <div className="content">{data}</div>;
}
```

**Angular Implementation:**

```typescript
// cls-prevention.component.ts
import { Component, OnInit } from '@angular/core';
import { onCLS } from 'web-vitals';

@Component({
  selector: 'app-cls-prevention',
  template: `
    <div class="container">
      <!-- Always specify image dimensions -->
      <img
        [src]="imageSrc"
        [attr.width]="imageWidth"
        [attr.height]="imageHeight"
        [alt]="imageAlt"
        class="responsive-image"
      />

      <!-- Use aspect ratio containers -->
      <div class="aspect-ratio-box">
        <img [src]="imageSrc" [alt]="imageAlt" class="aspect-ratio-content" />
      </div>

      <!-- Skeleton loading to reserve space -->
      <div *ngIf="loading" class="skeleton">
        <div class="skeleton-line"></div>
        <div class="skeleton-line"></div>
      </div>

      <div *ngIf="!loading" class="content">
        {{ content }}
      </div>
    </div>
  `,
  styles: [`
    .responsive-image {
      max-width: 100%;
      height: auto;
    }

    .aspect-ratio-box {
      position: relative;
      aspect-ratio: 16 / 9;
      width: 100%;
    }

    .aspect-ratio-content {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
    }

    .skeleton-line {
      height: 20px;
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 200% 100%;
      animation: loading 1.5s infinite;
      margin-bottom: 10px;
    }

    @keyframes loading {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class CLSPreventionComponent implements OnInit {
  imageSrc = '/assets/image.jpg';
  imageWidth = 800;
  imageHeight = 600;
  imageAlt = 'Description';
  loading = true;
  content = '';

  ngOnInit(): void {
    // Monitor CLS
    onCLS((metric) => {
      console.log('CLS value:', metric.value);
      
      if (metric.value > 0.1) {
        console.warn('Poor CLS detected!', metric.entries);
      }
    });

    // Simulate content loading
    setTimeout(() => {
      this.content = 'Loaded content';
      this.loading = false;
    }, 2000);
  }
}
```

## Additional Important Metrics

### Time to First Byte (TTFB)

TTFB measures the time from the request start to when the first byte of the response is received.

**Good TTFB**: ≤ 800ms
**Needs Improvement**: 800-1800ms
**Poor TTFB**: > 1800ms

```typescript
// Measuring TTFB
import { onTTFB } from 'web-vitals';

onTTFB((metric) => {
  console.log('TTFB:', metric.value);
  sendToAnalytics({
    name: 'TTFB',
    value: metric.value,
    rating: metric.rating
  });
});

// Optimizing TTFB with caching
// next.config.js (Next.js example)
module.exports = {
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, s-maxage=10, stale-while-revalidate=59'
          }
        ]
      }
    ];
  }
};

// React: Using service worker for better TTFB
// service-worker.js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      if (response) {
        return response;
      }
      return fetch(event.request).then((response) => {
        if (!response || response.status !== 200 || response.type !== 'basic') {
          return response;
        }
        const responseToCache = response.clone();
        caches.open('v1').then((cache) => {
          cache.put(event.request, responseToCache);
        });
        return response;
      });
    })
  );
});
```

### First Contentful Paint (FCP)

FCP measures when the first content is rendered on screen.

**Good FCP**: ≤ 1.8 seconds
**Needs Improvement**: 1.8-3.0 seconds
**Poor FCP**: > 3.0 seconds

```typescript
// Measuring FCP
import { onFCP } from 'web-vitals';

onFCP((metric) => {
  console.log('FCP:', metric.value);
  sendToAnalytics({
    name: 'FCP',
    value: metric.value,
    rating: metric.rating
  });
});

// Optimizing FCP
// 1. Inline critical CSS
function Document() {
  return (
    <html>
      <head>
        <style dangerouslySetInnerHTML={{
          __html: `
            /* Critical CSS */
            body { margin: 0; font-family: sans-serif; }
            .above-fold { display: block; }
          `
        }} />
        <link rel="stylesheet" href="/styles.css" media="print" onLoad="this.media='all'" />
      </head>
      <body>
        <div id="root"></div>
      </body>
    </html>
  );
}

// 2. Preload critical resources
function AppHead() {
  return (
    <head>
      <link rel="preload" href="/critical-font.woff2" as="font" type="font/woff2" crossOrigin="anonymous" />
      <link rel="preload" href="/hero-image.jpg" as="image" />
      <link rel="preconnect" href="https://api.example.com" />
    </head>
  );
}
```

## Comprehensive Monitoring System

```typescript
// web-vitals-monitor.ts
import { onCLS, onFCP, onINP, onLCP, onTTFB, Metric } from 'web-vitals';

interface VitalsData {
  CLS: number | null;
  FCP: number | null;
  INP: number | null;
  LCP: number | null;
  TTFB: number | null;
}

class WebVitalsMonitor {
  private vitals: VitalsData = {
    CLS: null,
    FCP: null,
    INP: null,
    LCP: null,
    TTFB: null
  };

  private listeners: ((vitals: VitalsData) => void)[] = [];

  startMonitoring(): void {
    onCLS(this.handleMetric.bind(this, 'CLS'));
    onFCP(this.handleMetric.bind(this, 'FCP'));
    onINP(this.handleMetric.bind(this, 'INP'));
    onLCP(this.handleMetric.bind(this, 'LCP'));
    onTTFB(this.handleMetric.bind(this, 'TTFB'));
  }

  private handleMetric(name: keyof VitalsData, metric: Metric): void {
    this.vitals[name] = metric.value;
    
    console.log(`${name}:`, {
      value: metric.value,
      rating: metric.rating,
      delta: metric.delta
    });

    // Send to analytics
    this.sendToAnalytics(name, metric);

    // Notify listeners
    this.notifyListeners();

    // Check for issues
    this.checkForIssues(name, metric);
  }

  private sendToAnalytics(name: string, metric: Metric): void {
    const analyticsData = {
      event: 'web_vitals',
      metric_name: name,
      value: Math.round(metric.value),
      rating: metric.rating,
      metric_id: metric.id,
      navigation_type: metric.navigationType,
      timestamp: Date.now(),
      url: window.location.href,
      user_agent: navigator.userAgent
    };

    // Send to Google Analytics
    if (typeof window !== 'undefined' && (window as any).gtag) {
      (window as any).gtag('event', name, analyticsData);
    }

    // Send to custom analytics endpoint
    fetch('/api/analytics', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(analyticsData),
      keepalive: true
    }).catch(console.error);
  }

  private checkForIssues(name: string, metric: Metric): void {
    if (metric.rating === 'poor') {
      console.warn(`⚠️ Poor ${name} detected:`, metric);

      // Log additional debug info
      if (name === 'LCP' && 'entries' in metric) {
        const lcpEntry = metric.entries[metric.entries.length - 1];
        console.log('LCP Element:', lcpEntry.element);
        console.log('LCP URL:', lcpEntry.url);
      }

      if (name === 'CLS' && 'entries' in metric) {
        console.log('Layout shift sources:', metric.entries);
      }

      // Send alert to monitoring service
      this.sendAlert(name, metric);
    }
  }

  private sendAlert(name: string, metric: Metric): void {
    // Send to error tracking service (e.g., Sentry)
    if (typeof window !== 'undefined' && (window as any).Sentry) {
      (window as any).Sentry.captureMessage(`Poor ${name} detected`, {
        level: 'warning',
        extra: {
          metric: name,
          value: metric.value,
          rating: metric.rating,
          url: window.location.href
        }
      });
    }
  }

  subscribe(listener: (vitals: VitalsData) => void): () => void {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }

  private notifyListeners(): void {
    this.listeners.forEach(listener => listener({ ...this.vitals }));
  }

  getVitals(): VitalsData {
    return { ...this.vitals };
  }

  getReport(): string {
    const ratings = {
      CLS: this.getRating('CLS', this.vitals.CLS, 0.1, 0.25),
      FCP: this.getRating('FCP', this.vitals.FCP, 1800, 3000),
      INP: this.getRating('INP', this.vitals.INP, 200, 500),
      LCP: this.getRating('LCP', this.vitals.LCP, 2500, 4000),
      TTFB: this.getRating('TTFB', this.vitals.TTFB, 800, 1800)
    };

    return `
Core Web Vitals Report
======================
LCP: ${this.vitals.LCP?.toFixed(0) || 'N/A'}ms [${ratings.LCP}]
INP: ${this.vitals.INP?.toFixed(0) || 'N/A'}ms [${ratings.INP}]
CLS: ${this.vitals.CLS?.toFixed(3) || 'N/A'} [${ratings.CLS}]

Additional Metrics
==================
FCP: ${this.vitals.FCP?.toFixed(0) || 'N/A'}ms [${ratings.FCP}]
TTFB: ${this.vitals.TTFB?.toFixed(0) || 'N/A'}ms [${ratings.TTFB}]
    `.trim();
  }

  private getRating(
    name: string,
    value: number | null,
    goodThreshold: number,
    poorThreshold: number
  ): string {
    if (value === null) return '⚪ Not measured';
    if (value <= goodThreshold) return '🟢 Good';
    if (value <= poorThreshold) return '🟡 Needs improvement';
    return '🔴 Poor';
  }
}

// Export singleton instance
export const webVitalsMonitor = new WebVitalsMonitor();

// React hook
export function useWebVitals() {
  const [vitals, setVitals] = useState<VitalsData>({
    CLS: null,
    FCP: null,
    INP: null,
    LCP: null,
    TTFB: null
  });

  useEffect(() => {
    webVitalsMonitor.startMonitoring();
    const unsubscribe = webVitalsMonitor.subscribe(setVitals);
    return unsubscribe;
  }, []);

  return vitals;
}

// Usage in React app
function PerformanceDashboard() {
  const vitals = useWebVitals();

  return (
    <div className="performance-dashboard">
      <h2>Core Web Vitals</h2>
      <div className="metrics">
        <Metric name="LCP" value={vitals.LCP} unit="ms" threshold={2500} />
        <Metric name="INP" value={vitals.INP} unit="ms" threshold={200} />
        <Metric name="CLS" value={vitals.CLS} unit="" threshold={0.1} />
        <Metric name="FCP" value={vitals.FCP} unit="ms" threshold={1800} />
        <Metric name="TTFB" value={vitals.TTFB} unit="ms" threshold={800} />
      </div>
    </div>
  );
}

function Metric({ name, value, unit, threshold }: any) {
  const rating = value === null ? 'unknown' :
                 value <= threshold ? 'good' :
                 value <= threshold * 1.6 ? 'needs-improvement' : 'poor';

  return (
    <div className={`metric metric-${rating}`}>
      <div className="metric-name">{name}</div>
      <div className="metric-value">
        {value === null ? 'N/A' : `${value.toFixed(0)}${unit}`}
      </div>
      <div className="metric-rating">{rating}</div>
    </div>
  );
}
```

**Angular Implementation:**

```typescript
// web-vitals.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { onCLS, onFCP, onINP, onLCP, onTTFB, Metric } from 'web-vitals';

interface VitalsData {
  CLS: number | null;
  FCP: number | null;
  INP: number | null;
  LCP: number | null;
  TTFB: number | null;
}

@Injectable({
  providedIn: 'root'
})
export class WebVitalsService {
  private vitalsSubject = new BehaviorSubject<VitalsData>({
    CLS: null,
    FCP: null,
    INP: null,
    LCP: null,
    TTFB: null
  });

  public vitals$: Observable<VitalsData> = this.vitalsSubject.asObservable();

  startMonitoring(): void {
    onCLS((metric) => this.handleMetric('CLS', metric));
    onFCP((metric) => this.handleMetric('FCP', metric));
    onINP((metric) => this.handleMetric('INP', metric));
    onLCP((metric) => this.handleMetric('LCP', metric));
    onTTFB((metric) => this.handleMetric('TTFB', metric));
  }

  private handleMetric(name: keyof VitalsData, metric: Metric): void {
    const currentVitals = this.vitalsSubject.value;
    this.vitalsSubject.next({
      ...currentVitals,
      [name]: metric.value
    });

    console.log(`${name}:`, metric);
    this.sendToAnalytics(name, metric);
  }

  private sendToAnalytics(name: string, metric: Metric): void {
    // Implementation similar to React version
  }

  getVitals(): VitalsData {
    return this.vitalsSubject.value;
  }
}

// performance-dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { WebVitalsService } from './web-vitals.service';

@Component({
  selector: 'app-performance-dashboard',
  template: `
    <div class="performance-dashboard">
      <h2>Core Web Vitals</h2>
      <div class="metrics">
        <div class="metric" *ngFor="let metric of metricsArray">
          <div class="metric-name">{{ metric.name }}</div>
          <div class="metric-value">
            {{ formatValue(metric.value, metric.unit) }}
          </div>
        </div>
      </div>
    </div>
  `
})
export class PerformanceDashboardComponent implements OnInit {
  metricsArray: any[] = [];

  constructor(private webVitals: WebVitalsService) {}

  ngOnInit(): void {
    this.webVitals.startMonitoring();
    
    this.webVitals.vitals$.subscribe(vitals => {
      this.metricsArray = [
        { name: 'LCP', value: vitals.LCP, unit: 'ms' },
        { name: 'INP', value: vitals.INP, unit: 'ms' },
        { name: 'CLS', value: vitals.CLS, unit: '' },
        { name: 'FCP', value: vitals.FCP, unit: 'ms' },
        { name: 'TTFB', value: vitals.TTFB, unit: 'ms' }
      ];
    });
  }

  formatValue(value: number | null, unit: string): string {
    return value === null ? 'N/A' : `${value.toFixed(0)}${unit}`;
  }
}
```

## Optimization Strategies ASCII Diagram

```
Core Web Vitals Optimization Flow
==================================

                    ┌─────────────────────────────┐
                    │   User Requests Page        │
                    └──────────┬──────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────┐
                    │   TTFB Optimization         │
                    │   - CDN                     │
                    │   - Server-side caching     │
                    │   - Optimize DB queries     │
                    └──────────┬──────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────┐
                    │   FCP Optimization          │
                    │   - Inline critical CSS     │
                    │   - Remove render-blocking  │
                    │   - Reduce JS execution     │
                    └──────────┬──────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────┐
                    │   LCP Optimization          │
                    │   - Optimize images         │
                    │   - Preload resources       │
                    │   - Code splitting          │
                    └──────────┬──────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────┐
                    │   CLS Optimization          │
                    │   - Set image dimensions    │
                    │   - Reserve space           │
                    │   - Use transforms          │
                    └──────────┬──────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────┐
                    │   INP Optimization          │
                    │   - Break up long tasks     │
                    │   - Debounce/throttle       │
                    │   - Use web workers         │
                    └──────────┬──────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────┐
                    │   Good User Experience      │
                    └─────────────────────────────┘
```

## Common Mistakes

1. **Not reserving space for dynamic content**
   - Problem: Causes layout shifts as content loads
   - Solution: Use skeleton screens or fixed heights

2. **Loading all JavaScript upfront**
   - Problem: Poor INP and LCP
   - Solution: Code splitting and lazy loading

3. **Not specifying image dimensions**
   - Problem: Major cause of CLS
   - Solution: Always include width and height attributes

4. **Ignoring font loading**
   - Problem: Font swaps cause CLS
   - Solution: Use font-display: swap and preload fonts

5. **Not measuring in production**
   - Problem: Lab data doesn't reflect real user experience
   - Solution: Implement RUM (Real User Monitoring)

6. **Optimizing for desktop only**
   - Problem: Mobile metrics are often worse
   - Solution: Test on actual devices and slow connections

7. **Animations causing layout shifts**
   - Problem: Non-transform animations trigger layout
   - Solution: Use transform and opacity only

8. **Third-party scripts blocking main thread**
   - Problem: Poor INP
   - Solution: Load third-party scripts asynchronously

## Best Practices

1. **Measure continuously**: Monitor Core Web Vitals in production with RUM
2. **Set performance budgets**: Define acceptable thresholds for each metric
3. **Test on real devices**: Lab conditions don't match real-world scenarios
4. **Optimize images**: Use modern formats, lazy loading, and proper dimensions
5. **Implement code splitting**: Load only what's needed for each route
6. **Use resource hints**: Preconnect, prefetch, and preload critical resources
7. **Minimize JavaScript**: Reduce bundle sizes and defer non-critical JS
8. **Reserve space for ads and embeds**: Prevent layout shifts from third-party content
9. **Optimize fonts**: Use font-display and preload critical fonts
10. **Use service workers**: Cache resources for faster repeat visits

## When to Use

**Core Web Vitals monitoring is essential when:**
- Building any user-facing web application
- SEO is important (Google uses these as ranking factors)
- User experience is a priority
- Performance optimization is needed
- A/B testing different implementations
- Debugging performance issues

**May be less critical when:**
- Building internal tools with no public traffic
- Working on APIs or backend services
- Creating proof-of-concept prototypes

## Interview Questions

1. **What are the three Core Web Vitals and what do they measure?**
   - LCP (Largest Contentful Paint): Loading performance
   - INP (Interaction to Next Paint): Interactivity
   - CLS (Cumulative Layout Shift): Visual stability

2. **How would you optimize LCP for a page with large hero images?**
   - Use responsive images with srcset
   - Implement lazy loading for below-the-fold images
   - Preload the hero image
   - Use modern image formats (WebP, AVIF)
   - Optimize image compression
   - Use a CDN

3. **What causes poor INP and how do you fix it?**
   - Causes: Long JavaScript tasks, synchronous operations, expensive re-renders
   - Fixes: Break up long tasks, use debouncing/throttling, implement code splitting, use web workers for heavy computation

4. **Explain common causes of CLS and prevention strategies.**
   - Causes: Images without dimensions, dynamic content insertion, web fonts, ads/embeds
   - Prevention: Set explicit dimensions, reserve space, use font-display, implement skeleton screens

5. **How do you implement Core Web Vitals monitoring in a production app?**
   - Use web-vitals library
   - Send metrics to analytics
   - Implement RUM (Real User Monitoring)
   - Set up alerts for poor metrics
   - Create dashboards for visualization

6. **What's the difference between lab data and field data for Core Web Vitals?**
   - Lab data: Controlled environment (Lighthouse), consistent, good for debugging
   - Field data: Real users (CrUX), variable conditions, reflects actual experience
   - Both needed for complete picture

7. **How does code splitting affect Core Web Vitals?**
   - Improves LCP by reducing initial bundle size
   - Can improve INP by reducing main thread work
   - Must be implemented carefully to avoid waterfalls
   - Route-based splitting is most effective

8. **What role do service workers play in Core Web Vitals optimization?**
   - Improve TTFB through caching
   - Enable offline functionality
   - Reduce network requests
   - Can implement stale-while-revalidate pattern
   - Help with repeat visit performance

## Key Takeaways

1. Core Web Vitals consist of three main metrics: LCP (loading), INP (interactivity), and CLS (visual stability)
2. Google uses Core Web Vitals as ranking factors for search results
3. Good thresholds: LCP ≤2.5s, INP ≤200ms, CLS ≤0.1
4. Always specify image dimensions to prevent layout shifts
5. Implement code splitting and lazy loading to improve LCP and INP
6. Use the web-vitals library for accurate measurement
7. Monitor both lab data (Lighthouse) and field data (RUM) for complete insights
8. Optimize for mobile first, as mobile metrics are typically worse
9. Set performance budgets and monitor continuously in production
10. Prevention is easier than fixing: build with performance in mind from the start

## Resources

- **Official Documentation**
  - [Web Vitals Documentation](https://web.dev/vitals/)
  - [Chrome User Experience Report](https://developers.google.com/web/tools/chrome-user-experience-report)
  - [web-vitals Library](https://github.com/GoogleChrome/web-vitals)

- **Tools**
  - [PageSpeed Insights](https://pagespeed.web.dev/)
  - [Lighthouse](https://developers.google.com/web/tools/lighthouse)
  - [WebPageTest](https://www.webpagetest.org/)
  - [Chrome DevTools Performance Panel](https://developer.chrome.com/docs/devtools/performance/)

- **Articles**
  - [Optimizing Core Web Vitals](https://web.dev/optimize-cwv-tools/)
  - [INP Optimization Guide](https://web.dev/optimize-inp/)
  - [CLS Debugging Guide](https://web.dev/debug-layout-shifts/)

- **Courses**
  - [Web.dev Learn Performance](https://web.dev/learn/performance/)
  - [Frontend Masters Web Performance](https://frontendmasters.com/courses/web-performance/)
