# Performance Monitoring

## The Idea

**In plain English:** Performance monitoring means keeping a constant eye on how fast (or slow) your website feels to real people who visit it, collecting numbers like "how long did it take before something appeared on screen?" so you can spot and fix slowdowns before they drive users away.

**Real-world analogy:** Think of a hospital's patient-monitoring system. Nurses attach sensors to patients that continuously track heart rate, blood pressure, and oxygen levels, and an alarm goes off the moment any reading goes outside the safe range.

- The sensors attached to each patient = the tracking code (PerformanceObserver, web-vitals library) embedded in your webpage
- The vital-sign readings (heart rate, blood pressure) = the performance metrics (LCP, FID, CLS) being measured
- The central nurses' station dashboard = the analytics platform or custom dashboard that collects and displays all the data
- The alarm that fires when a reading is dangerous = the alerting system that notifies your team when a metric exceeds its threshold

---

## Overview

Performance monitoring tracks real-world performance metrics to identify issues, regressions, and optimization opportunities. Real User Monitoring (RUM) collects data from actual users, while synthetic monitoring tests from controlled environments. This guide covers RUM implementation, synthetic monitoring strategies, web-vitals library integration, analytics platforms, and alerting systems.

## Real User Monitoring (RUM)

RUM collects performance data from actual users in production:

```
RUM vs Synthetic Monitoring:

RUM (Real User Monitoring):
┌────────────────────────────────────────────────────┐
│  Actual Users → Real Devices → Real Networks       │
│  - True performance data                           │
│  - Geographic distribution                         │
│  - Device/browser variety                          │
│  - Network conditions (3G, 4G, WiFi)              │
│  - User behavior patterns                          │
└────────────────────────────────────────────────────┘

Synthetic Monitoring:
┌────────────────────────────────────────────────────┐
│  Bot/Script → Controlled Environment               │
│  - Consistent baseline                             │
│  - Pre-deployment testing                          │
│  - Controlled conditions                           │
│  - Scheduled checks                                │
│  - Regression detection                            │
└────────────────────────────────────────────────────┘
```

### Basic RUM Implementation

```typescript
// PerformanceMonitor.ts
export class PerformanceMonitor {
  private metrics: Map<string, number> = new Map();
  private apiEndpoint: string;
  
  constructor(apiEndpoint: string) {
    this.apiEndpoint = apiEndpoint;
    this.initObservers();
    this.captureNavigationTiming();
    this.captureResourceTiming();
  }
  
  private initObservers(): void {
    // Observe paint timing
    if ('PerformanceObserver' in window) {
      // FCP, LCP observer
      const paintObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          if (entry.name === 'first-contentful-paint') {
            this.metrics.set('fcp', entry.startTime);
          }
          
          if (entry.entryType === 'largest-contentful-paint') {
            this.metrics.set('lcp', entry.startTime);
          }
        }
      });
      
      paintObserver.observe({
        entryTypes: ['paint', 'largest-contentful-paint']
      });
      
      // FID observer
      const fidObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          this.metrics.set('fid', (entry as any).processingStart - entry.startTime);
        }
      });
      
      fidObserver.observe({ entryTypes: ['first-input'] });
      
      // Long tasks observer
      const longTaskObserver = new PerformanceObserver((list) => {
        const count = this.metrics.get('longTasks') || 0;
        this.metrics.set('longTasks', count + list.getEntries().length);
      });
      
      longTaskObserver.observe({ entryTypes: ['longtask'] });
    }
  }
  
  private captureNavigationTiming(): void {
    window.addEventListener('load', () => {
      setTimeout(() => {
        const navigation = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
        
        if (navigation) {
          this.metrics.set('dns', navigation.domainLookupEnd - navigation.domainLookupStart);
          this.metrics.set('tcp', navigation.connectEnd - navigation.connectStart);
          this.metrics.set('ttfb', navigation.responseStart - navigation.requestStart);
          this.metrics.set('domLoad', navigation.domContentLoadedEventEnd - navigation.domContentLoadedEventStart);
          this.metrics.set('windowLoad', navigation.loadEventEnd - navigation.loadEventStart);
        }
      }, 0);
    });
  }
  
  private captureResourceTiming(): void {
    window.addEventListener('load', () => {
      setTimeout(() => {
        const resources = performance.getEntriesByType('resource') as PerformanceResourceTiming[];
        
        const resourceMetrics = {
          totalResources: resources.length,
          totalSize: 0,
          totalDuration: 0,
          scripts: 0,
          styles: 0,
          images: 0,
          fonts: 0
        };
        
        resources.forEach(resource => {
          resourceMetrics.totalSize += resource.transferSize || 0;
          resourceMetrics.totalDuration += resource.duration;
          
          if (resource.initiatorType === 'script') resourceMetrics.scripts++;
          if (resource.initiatorType === 'link' || resource.initiatorType === 'css') resourceMetrics.styles++;
          if (resource.initiatorType === 'img') resourceMetrics.images++;
          if (resource.initiatorType === 'font') resourceMetrics.fonts++;
        });
        
        Object.entries(resourceMetrics).forEach(([key, value]) => {
          this.metrics.set(key, value);
        });
      }, 0);
    });
  }
  
  async send(): Promise<void> {
    const data = {
      url: window.location.href,
      userAgent: navigator.userAgent,
      metrics: Object.fromEntries(this.metrics),
      connection: (navigator as any).connection ? {
        effectiveType: (navigator as any).connection.effectiveType,
        downlink: (navigator as any).connection.downlink,
        rtt: (navigator as any).connection.rtt
      } : null,
      memory: (performance as any).memory ? {
        usedJSHeapSize: (performance as any).memory.usedJSHeapSize,
        totalJSHeapSize: (performance as any).memory.totalJSHeapSize,
        jsHeapSizeLimit: (performance as any).memory.jsHeapSizeLimit
      } : null,
      timestamp: Date.now()
    };
    
    try {
      // Use sendBeacon for reliability
      if (navigator.sendBeacon) {
        navigator.sendBeacon(
          this.apiEndpoint,
          JSON.stringify(data)
        );
      } else {
        await fetch(this.apiEndpoint, {
          method: 'POST',
          body: JSON.stringify(data),
          headers: { 'Content-Type': 'application/json' },
          keepalive: true
        });
      }
    } catch (error) {
      console.error('Failed to send metrics:', error);
    }
  }
}

// Initialize
const monitor = new PerformanceMonitor('https://api.example.com/metrics');

// Send metrics before page unload
window.addEventListener('pagehide', () => {
  monitor.send();
});
```

### React RUM Hook

```typescript
// usePerformanceMonitoring.ts
import { useEffect, useRef } from 'react';

interface PerformanceMetrics {
  fcp?: number;
  lcp?: number;
  fid?: number;
  cls?: number;
  ttfb?: number;
}

export function usePerformanceMonitoring(
  endpoint: string,
  options: {
    sampleRate?: number;
    debounceMs?: number;
  } = {}
) {
  const { sampleRate = 1, debounceMs = 5000 } = options;
  const metrics = useRef<PerformanceMetrics>({});
  const timeoutId = useRef<NodeJS.Timeout>();
  
  useEffect(() => {
    // Sample rate check
    if (Math.random() > sampleRate) return;
    
    const sendMetrics = () => {
      if (Object.keys(metrics.current).length === 0) return;
      
      const data = {
        url: window.location.pathname,
        metrics: metrics.current,
        userAgent: navigator.userAgent,
        timestamp: Date.now()
      };
      
      navigator.sendBeacon(endpoint, JSON.stringify(data));
    };
    
    const scheduleSend = () => {
      if (timeoutId.current) {
        clearTimeout(timeoutId.current);
      }
      
      timeoutId.current = setTimeout(sendMetrics, debounceMs);
    };
    
    // Observe Web Vitals
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        switch (entry.entryType) {
          case 'paint':
            if (entry.name === 'first-contentful-paint') {
              metrics.current.fcp = entry.startTime;
              scheduleSend();
            }
            break;
            
          case 'largest-contentful-paint':
            metrics.current.lcp = entry.startTime;
            scheduleSend();
            break;
            
          case 'first-input':
            metrics.current.fid = (entry as any).processingStart - entry.startTime;
            scheduleSend();
            break;
            
          case 'layout-shift':
            if (!(entry as any).hadRecentInput) {
              metrics.current.cls = (metrics.current.cls || 0) + (entry as any).value;
              scheduleSend();
            }
            break;
        }
      }
    });
    
    observer.observe({
      entryTypes: ['paint', 'largest-contentful-paint', 'first-input', 'layout-shift']
    });
    
    // Send on page hide
    const handlePageHide = () => sendMetrics();
    window.addEventListener('pagehide', handlePageHide);
    
    return () => {
      observer.disconnect();
      window.removeEventListener('pagehide', handlePageHide);
      if (timeoutId.current) {
        clearTimeout(timeoutId.current);
      }
    };
  }, [endpoint, sampleRate, debounceMs]);
}

// Usage
function App() {
  usePerformanceMonitoring('https://api.example.com/metrics', {
    sampleRate: 0.1, // Sample 10% of users
    debounceMs: 5000
  });
  
  return <div>App content</div>;
}
```

### Angular RUM Service

```typescript
// performance-monitoring.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

interface PerformanceData {
  route: string;
  metrics: Record<string, number>;
  timestamp: number;
}

@Injectable({
  providedIn: 'root'
})
export class PerformanceMonitoringService implements OnDestroy {
  private observers: PerformanceObserver[] = [];
  private metrics: Map<string, number> = new Map();
  private apiEndpoint = 'https://api.example.com/metrics';
  
  constructor(private router: Router) {
    this.initObservers();
    this.trackRouteChanges();
  }
  
  private initObservers(): void {
    if ('PerformanceObserver' in window) {
      // Web Vitals observer
      const vitalsObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          this.handlePerformanceEntry(entry);
        }
      });
      
      vitalsObserver.observe({
        entryTypes: ['paint', 'largest-contentful-paint', 'first-input', 'layout-shift']
      });
      
      this.observers.push(vitalsObserver);
    }
  }
  
  private handlePerformanceEntry(entry: PerformanceEntry): void {
    switch (entry.entryType) {
      case 'paint':
        if (entry.name === 'first-contentful-paint') {
          this.metrics.set('fcp', entry.startTime);
        }
        break;
        
      case 'largest-contentful-paint':
        this.metrics.set('lcp', entry.startTime);
        break;
        
      case 'first-input':
        this.metrics.set('fid', (entry as any).processingStart - entry.startTime);
        break;
        
      case 'layout-shift':
        if (!(entry as any).hadRecentInput) {
          const currentCLS = this.metrics.get('cls') || 0;
          this.metrics.set('cls', currentCLS + (entry as any).value);
        }
        break;
    }
  }
  
  private trackRouteChanges(): void {
    this.router.events
      .pipe(filter(event => event instanceof NavigationEnd))
      .subscribe((event: NavigationEnd) => {
        // Send metrics for previous route
        this.sendMetrics();
        
        // Clear metrics for new route
        this.metrics.clear();
      });
  }
  
  private sendMetrics(): void {
    if (this.metrics.size === 0) return;
    
    const data: PerformanceData = {
      route: this.router.url,
      metrics: Object.fromEntries(this.metrics),
      timestamp: Date.now()
    };
    
    if (navigator.sendBeacon) {
      navigator.sendBeacon(this.apiEndpoint, JSON.stringify(data));
    }
  }
  
  ngOnDestroy(): void {
    this.observers.forEach(observer => observer.disconnect());
    this.sendMetrics();
  }
}

// app.component.ts
@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent {
  constructor(private perfMonitoring: PerformanceMonitoringService) {
    // Service automatically starts monitoring
  }
}
```

## Web Vitals Library

Google's web-vitals library simplifies Core Web Vitals tracking:

```bash
npm install web-vitals
```

### Basic Integration

```typescript
// metrics.ts
import {
  onCLS,
  onFID,
  onFCP,
  onLCP,
  onTTFB,
  onINP,
  Metric
} from 'web-vitals';

function sendToAnalytics(metric: Metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    navigationType: metric.navigationType,
    url: window.location.href,
    timestamp: Date.now()
  });
  
  // Use sendBeacon for reliability
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/analytics', body);
  } else {
    fetch('/analytics', {
      method: 'POST',
      body,
      keepalive: true
    });
  }
}

// Track all Web Vitals
onCLS(sendToAnalytics);
onFID(sendToAnalytics);
onFCP(sendToAnalytics);
onLCP(sendToAnalytics);
onTTFB(sendToAnalytics);
onINP(sendToAnalytics); // New metric replacing FID
```

### React Web Vitals Integration

```typescript
// useWebVitals.ts
import { useEffect } from 'react';
import { onCLS, onFID, onFCP, onLCP, onTTFB, Metric } from 'web-vitals';

export function useWebVitals(
  onMetric: (metric: Metric) => void,
  options: { reportAllChanges?: boolean } = {}
) {
  useEffect(() => {
    onCLS(onMetric, options);
    onFID(onMetric, options);
    onFCP(onMetric, options);
    onLCP(onMetric, options);
    onTTFB(onMetric, options);
  }, [onMetric, options]);
}

// App.tsx
import { useWebVitals } from './useWebVitals';

function App() {
  useWebVitals((metric) => {
    // Send to analytics
    console.log(metric);
    
    // Track in Google Analytics
    if (window.gtag) {
      window.gtag('event', metric.name, {
        value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
        event_category: 'Web Vitals',
        event_label: metric.id,
        non_interaction: true
      });
    }
    
    // Track in custom analytics
    fetch('/api/metrics', {
      method: 'POST',
      body: JSON.stringify(metric),
      keepalive: true
    });
  }, { reportAllChanges: true });
  
  return <div>App content</div>;
}
```

### Angular Web Vitals Service

```typescript
// web-vitals.service.ts
import { Injectable } from '@angular/core';
import { onCLS, onFID, onFCP, onLCP, onTTFB, Metric } from 'web-vitals';

@Injectable({
  providedIn: 'root'
})
export class WebVitalsService {
  private apiEndpoint = '/api/metrics';
  
  init(): void {
    const handleMetric = (metric: Metric) => {
      this.sendMetric(metric);
    };
    
    onCLS(handleMetric);
    onFID(handleMetric);
    onFCP(handleMetric);
    onLCP(handleMetric);
    onTTFB(handleMetric);
  }
  
  private sendMetric(metric: Metric): void {
    const data = {
      name: metric.name,
      value: metric.value,
      rating: metric.rating,
      url: window.location.href,
      timestamp: Date.now()
    };
    
    navigator.sendBeacon(this.apiEndpoint, JSON.stringify(data));
  }
}

// app.component.ts
@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent implements OnInit {
  constructor(private webVitals: WebVitalsService) {}
  
  ngOnInit(): void {
    this.webVitals.init();
  }
}
```

## Synthetic Monitoring

Automated testing from controlled environments:

### Lighthouse CI

```bash
# Install Lighthouse CI
npm install -g @lhci/cli

# lighthouserc.json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:3000"],
      "numberOfRuns": 3
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "categories:performance": ["error", {"minScore": 0.9}],
        "categories:accessibility": ["error", {"minScore": 0.9}],
        "first-contentful-paint": ["error", {"maxNumericValue": 2000}],
        "largest-contentful-paint": ["error", {"maxNumericValue": 2500}],
        "cumulative-layout-shift": ["error", {"maxNumericValue": 0.1}],
        "total-blocking-time": ["error", {"maxNumericValue": 300}]
      }
    },
    "upload": {
      "target": "temporary-public-storage"
    }
  }
}

# Run in CI/CD
lhci autorun
```

### GitHub Actions Integration

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

### Puppeteer Performance Testing

```typescript
// performance-test.ts
import puppeteer from 'puppeteer';

async function runPerformanceTest(url: string) {
  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();
  
  // Enable performance monitoring
  await page.evaluateOnNewDocument(() => {
    window.performance.mark('test-start');
  });
  
  // Navigate and wait for load
  await page.goto(url, { waitUntil: 'networkidle0' });
  
  // Collect metrics
  const metrics = await page.evaluate(() => {
    const navigation = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
    const paint = performance.getEntriesByType('paint');
    
    return {
      ttfb: navigation.responseStart - navigation.requestStart,
      domLoad: navigation.domContentLoadedEventEnd - navigation.fetchStart,
      windowLoad: navigation.loadEventEnd - navigation.fetchStart,
      fcp: paint.find(entry => entry.name === 'first-contentful-paint')?.startTime,
      resources: performance.getEntriesByType('resource').length,
      memory: (performance as any).memory ? {
        used: (performance as any).memory.usedJSHeapSize,
        total: (performance as any).memory.totalJSHeapSize
      } : null
    };
  });
  
  await browser.close();
  
  return metrics;
}

// Run test
runPerformanceTest('https://example.com').then(metrics => {
  console.log('Performance Metrics:', metrics);
  
  // Assert thresholds
  if (metrics.ttfb > 600) {
    throw new Error(`TTFB too high: ${metrics.ttfb}ms`);
  }
  
  if (metrics.fcp > 2000) {
    throw new Error(`FCP too high: ${metrics.fcp}ms`);
  }
});
```

## Analytics Integration

### Google Analytics 4

```typescript
// Send Web Vitals to GA4
import { onCLS, onFID, onLCP, Metric } from 'web-vitals';

function sendToGA4(metric: Metric) {
  if (window.gtag) {
    window.gtag('event', metric.name, {
      value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
      metric_id: metric.id,
      metric_value: metric.value,
      metric_delta: metric.delta,
      metric_rating: metric.rating
    });
  }
}

onCLS(sendToGA4);
onFID(sendToGA4);
onLCP(sendToGA4);
```

### Custom Dashboard

```typescript
// performance-dashboard.tsx
import React, { useEffect, useState } from 'react';
import { Line } from 'react-chartjs-2';

interface MetricData {
  timestamp: number;
  value: number;
}

interface PerformanceDashboard {
  fcp: MetricData[];
  lcp: MetricData[];
  fid: MetricData[];
  cls: MetricData[];
}

export const PerformanceDashboard: React.FC = () => {
  const [data, setData] = useState<PerformanceDashboard | null>(null);
  const [timeRange, setTimeRange] = useState<'1h' | '24h' | '7d'>('24h');
  
  useEffect(() => {
    fetch(`/api/metrics?range=${timeRange}`)
      .then(res => res.json())
      .then(setData);
  }, [timeRange]);
  
  if (!data) return <div>Loading...</div>;
  
  return (
    <div className="performance-dashboard">
      <h1>Performance Metrics</h1>
      
      <div className="time-range-selector">
        <button onClick={() => setTimeRange('1h')}>1 Hour</button>
        <button onClick={() => setTimeRange('24h')}>24 Hours</button>
        <button onClick={() => setTimeRange('7d')}>7 Days</button>
      </div>
      
      <div className="metrics-grid">
        <MetricCard
          title="First Contentful Paint"
          data={data.fcp}
          threshold={1800}
          unit="ms"
        />
        
        <MetricCard
          title="Largest Contentful Paint"
          data={data.lcp}
          threshold={2500}
          unit="ms"
        />
        
        <MetricCard
          title="First Input Delay"
          data={data.fid}
          threshold={100}
          unit="ms"
        />
        
        <MetricCard
          title="Cumulative Layout Shift"
          data={data.cls}
          threshold={0.1}
          unit=""
        />
      </div>
    </div>
  );
};

const MetricCard: React.FC<{
  title: string;
  data: MetricData[];
  threshold: number;
  unit: string;
}> = ({ title, data, threshold, unit }) => {
  const latest = data[data.length - 1]?.value || 0;
  const isGood = latest <= threshold;
  
  const chartData = {
    labels: data.map(d => new Date(d.timestamp).toLocaleTimeString()),
    datasets: [{
      label: title,
      data: data.map(d => d.value),
      borderColor: isGood ? '#10b981' : '#ef4444',
      tension: 0.1
    }]
  };
  
  return (
    <div className={`metric-card metric-card--${isGood ? 'good' : 'bad'}`}>
      <h3>{title}</h3>
      <div className="metric-value">
        {latest.toFixed(2)} {unit}
      </div>
      <div className="metric-status">
        {isGood ? '✓ Good' : '✗ Needs Improvement'}
      </div>
      <Line data={chartData} />
    </div>
  );
};
```

## Common Mistakes

### 1. Not Sampling in Production

```typescript
// ❌ Bad: Track all users (performance overhead)
onLCP(sendToAnalytics);

// ✅ Good: Sample 10% of users
if (Math.random() < 0.1) {
  onLCP(sendToAnalytics);
}
```

### 2. Sending Too Many Events

```typescript
// ❌ Bad: Send every metric update
onLCP((metric) => {
  fetch('/analytics', { method: 'POST', body: JSON.stringify(metric) });
}, { reportAllChanges: true });

// ✅ Good: Debounce and batch
const batch: Metric[] = [];
onLCP((metric) => {
  batch.push(metric);
  debouncedSend(batch);
});
```

### 3. Not Using sendBeacon

```typescript
// ❌ Bad: Regular fetch (may not complete on page unload)
window.addEventListener('unload', () => {
  fetch('/analytics', { method: 'POST', body: data });
});

// ✅ Good: Use sendBeacon (reliable)
window.addEventListener('pagehide', () => {
  navigator.sendBeacon('/analytics', data);
});
```

### 4. Ignoring Network Conditions

```typescript
// ❌ Bad: Same tracking for all connections
onLCP(sendToAnalytics);

// ✅ Good: Adjust based on connection
const connection = (navigator as any).connection;
const sampleRate = connection?.effectiveType === '4g' ? 0.1 : 0.05;

if (Math.random() < sampleRate) {
  onLCP(sendToAnalytics);
}
```

### 5. No Alerting

```typescript
// ❌ Bad: Just collect metrics
onLCP(sendToAnalytics);

// ✅ Good: Alert on thresholds
onLCP((metric) => {
  sendToAnalytics(metric);
  
  if (metric.value > 2500) {
    alertTeam({
      metric: 'LCP',
      value: metric.value,
      threshold: 2500,
      url: window.location.href
    });
  }
});
```

## Best Practices

1. **Sample users in production** - 5-10% is usually sufficient
2. **Use sendBeacon** - reliable data transmission on page unload
3. **Track Core Web Vitals** - FCP, LCP, FID/INP, CLS, TTFB
4. **Segment by demographics** - device, connection, geography
5. **Set up alerts** - notify when metrics exceed thresholds
6. **Combine RUM and synthetic** - real users + baseline testing
7. **Track custom metrics** - business-specific performance indicators
8. **Batch and debounce** - reduce analytics overhead
9. **Monitor in CI/CD** - Lighthouse CI for regression detection
10. **Create dashboards** - visualize trends and identify issues

## When to Use

### RUM
- Production monitoring
- Understanding real-world performance
- Geographic/device analysis
- User behavior correlation

### Synthetic Monitoring
- Pre-deployment testing
- Regression detection
- Baseline establishment
- Controlled comparisons

### Both
- Comprehensive monitoring strategy
- Compare controlled vs real-world
- Validate optimizations

## Interview Questions

### 1. What is the difference between RUM and synthetic monitoring?

**Answer**: RUM (Real User Monitoring) collects performance data from actual users in production, providing real-world metrics across diverse devices, networks, and geographies. Synthetic monitoring uses bots/scripts in controlled environments to test performance consistently, detecting regressions and establishing baselines. RUM shows true user experience but with variability; synthetic provides consistent data for comparison. Best practice: use both - synthetic for pre-deployment testing, RUM for production monitoring.

### 2. Why should you use navigator.sendBeacon for analytics?

**Answer**: `sendBeacon` is designed for reliability when sending data during page unload. Regular fetch/XHR requests may be cancelled when the page unloads, losing data. sendBeacon queues data for asynchronous transmission even after page closure, with no impact on navigation performance. It's the recommended method for analytics, especially for measuring page abandonment and exit metrics. Fallback to fetch with keepalive:true for older browsers.

### 3. What sample rate should you use for RUM in production?

**Answer**: Typically 5-10% for most sites, adjusting based on traffic volume. High-traffic sites (millions of users) can sample 1-5%, while lower-traffic sites might sample 20-50% for statistical significance. Consider: (1) Analytics cost (storage, processing), (2) Statistical significance (need enough data), (3) Performance overhead (tracking code execution), (4) Network conditions (sample less on slow connections). Always sample deterministically (based on user ID) to track same users consistently.

### 4. How do you track single-page application (SPA) performance?

**Answer**: Traditional metrics (page load) only apply to initial load. For SPAs: (1) Track route changes with custom marks, (2) Measure route transition time, (3) Track LCP per route, (4) Monitor INP for interactions, (5) Use PerformanceMark/Measure API, (6) Track component mount times, (7) Monitor data fetching time. Use frameworks' built-in tools (React Profiler, Angular DevTools) and custom instrumentation. Reset/re-measure Core Web Vitals on route changes.

### 5. What are the Core Web Vitals thresholds?

**Answer**:
- LCP (Largest Contentful Paint): Good <2.5s, Needs Improvement 2.5-4s, Poor >4s
- FID/INP (First Input Delay/Interaction to Next Paint): Good <100ms/200ms, Needs Improvement 100-300ms/200-500ms, Poor >300ms/500ms
- CLS (Cumulative Layout Shift): Good <0.1, Needs Improvement 0.1-0.25, Poor >0.25
Measured at 75th percentile of page loads (must be good for 75% of users).

### 6. How would you implement performance regression detection in CI/CD?

**Answer**: 
1. Use Lighthouse CI to run performance audits on every PR
2. Set budget thresholds in lighthouserc.json (FCP <2s, LCP <2.5s, etc.)
3. Compare against baseline branch (main)
4. Fail build if metrics regress beyond threshold (e.g., >10% worse)
5. Comment on PR with results and comparison
6. Store historical data to track trends
7. Consider multiple runs (3-5) for reliability
8. Test on representative device/network profiles

### 7. What is the web-vitals library and why use it?

**Answer**: web-vitals is Google's official library for measuring Core Web Vitals (LCP, FID, CLS, FCP, TTFB, INP). Benefits: (1) Handles browser API complexity, (2) Consistent measurement methodology, (3) Matches Chrome User Experience Report data, (4) Actively maintained by Chrome team, (5) Small size (~1KB gzipped), (6) Works with all analytics platforms. It abstracts PerformanceObserver APIs and provides reliable, standardized measurements aligned with how Google measures for rankings.

### 8. How do you correlate performance metrics with business metrics?

**Answer**: 
1. Track conversions alongside performance (e.g., checkout completion vs LCP)
2. Segment users by performance (fast vs slow experience)
3. Measure bounce rate correlation with metrics
4. A/B test performance optimizations against business KPIs
5. Calculate revenue impact per 100ms improvement
6. Track engagement metrics (time on site, pages per session) by performance bucket
Studies show: 100ms LCP improvement = ~1% conversion increase for e-commerce.

## Key Takeaways

1. **RUM provides real-world data** - actual users, devices, networks, and geographies
2. **Synthetic monitoring establishes baselines** - controlled testing for regression detection
3. **Use web-vitals library** - standardized Core Web Vitals measurement
4. **Sample users appropriately** - 5-10% is sufficient for most sites
5. **sendBeacon for reliability** - ensures analytics data reaches server
6. **Track Core Web Vitals** - LCP, FID/INP, CLS, FCP, TTFB at 75th percentile
7. **Set up alerting** - notify team when metrics exceed thresholds
8. **Integrate with CI/CD** - Lighthouse CI for pre-deployment testing
9. **Segment and analyze** - by device, connection, geography, user type
10. **Correlate with business metrics** - measure performance impact on conversions

## Resources

- [Web Vitals (Google)](https://web.dev/vitals/)
- [web-vitals Library](https://github.com/GoogleChrome/web-vitals)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [Chrome User Experience Report](https://developers.google.com/web/tools/chrome-user-experience-report)
- [Performance Observer (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver)
- [sendBeacon (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon)
- [Real User Monitoring (Cloudflare)](https://www.cloudflare.com/learning/performance/what-is-real-user-monitoring/)
