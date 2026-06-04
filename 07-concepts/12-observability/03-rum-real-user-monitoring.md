# RUM — Real User Monitoring

## Overview

Real User Monitoring (RUM) collects performance and behavior metrics from actual users' browsers as they interact with your application. Unlike synthetic monitoring (running scripted tests in a controlled lab environment), RUM captures the true distribution of experiences across different devices, networks, locations, and browser versions. This guide covers the difference between RUM and synthetic monitoring, using the PerformanceObserver API to collect Core Web Vitals, reporting strategies, and privacy considerations.

## RUM vs Synthetic Monitoring

```
Synthetic Monitoring:
  ┌──────────────────────────────────────────────┐
  │  Controlled lab environment                  │
  │  - Fixed device (Moto G4 emulation)          │
  │  - Fixed network (throttled 4G)              │
  │  - Scripted journey                          │
  │  - Deterministic, reproducible               │
  │  - Runs on schedule (every 5 min)            │
  │  - Zero real users needed                    │
  │  Tools: Lighthouse, WebPageTest, Pingdom     │
  └──────────────────────────────────────────────┘

Real User Monitoring (RUM):
  ┌──────────────────────────────────────────────┐
  │  Real users, real devices, real networks     │
  │  - Mix of iPhone 15 and Galaxy A12           │
  │  - 5G, LTE, WiFi, 3G — whatever users have  │
  │  - Actual navigation paths                   │
  │  - Probabilistic, statistical                │
  │  - Continuous (every user session)           │
  │  - Requires traffic to produce data          │
  │  Tools: web-vitals, Datadog RUM, Sentry,     │
  │         New Relic, custom PerformanceObserver│
  └──────────────────────────────────────────────┘

When each is appropriate:
  Synthetic: CI performance regression detection, pre-launch baseline
  RUM: understanding real user experience, tracking p75/p95, A/B perf testing
  Best: BOTH — synthetic catches regressions, RUM validates with real users
```

## Core Web Vitals — The Metrics That Matter

```
Core Web Vitals (Google's field metrics):

LCP — Largest Contentful Paint
  What: Time until the largest content element renders
  Threshold: ≤ 2.5s = Good, > 4s = Poor
  Captures: Loading performance

INP — Interaction to Next Paint (replaced FID in 2024)
  What: Worst-case latency for user interactions (click, tap, keypress)
  Threshold: ≤ 200ms = Good, > 500ms = Poor
  Captures: Responsiveness during interaction

CLS — Cumulative Layout Shift
  What: Sum of layout shift scores from unexpected visual shifts
  Threshold: ≤ 0.1 = Good, > 0.25 = Poor
  Captures: Visual stability

Supporting metrics (not CWV but tracked):
  TTFB — Time to First Byte
  FCP — First Contentful Paint
  TBT — Total Blocking Time (lab only)
```

## PerformanceObserver API

The browser-native API for observing performance events:

```typescript
// Observe Long Animation Frames (replaces LoAF in modern browsers)
const longAnimationFrameObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const loaf = entry as PerformanceLongAnimationFrameTiming;
    console.log(`Long animation frame: ${loaf.duration}ms`);
  }
});
longAnimationFrameObserver.observe({ type: 'long-animation-frame', buffered: true });

// Observe Long Tasks (older API)
const longTaskObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`Long task: ${entry.duration}ms`, entry);
  }
});
longTaskObserver.observe({ type: 'longtask', buffered: true });

// Observe Navigation timing
const navigationObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const nav = entry as PerformanceNavigationTiming;
    console.log({
      ttfb: nav.responseStart - nav.fetchStart,
      fcp: nav.domContentLoadedEventEnd - nav.fetchStart,
      domLoad: nav.domContentLoadedEventEnd - nav.fetchStart,
      totalLoad: nav.loadEventEnd - nav.fetchStart,
    });
  }
});
navigationObserver.observe({ type: 'navigation', buffered: true });

// Observe Paint timing (FCP)
const paintObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name === 'first-contentful-paint') {
      console.log(`FCP: ${entry.startTime}ms`);
    }
  }
});
paintObserver.observe({ type: 'paint', buffered: true });

// Observe Largest Contentful Paint
const lcpObserver = new PerformanceObserver((list) => {
  // LCP may be updated as more content renders — use last entry
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1] as LargestContentfulPaint;
  console.log(`LCP candidate: ${lastEntry.startTime}ms`, {
    element: lastEntry.element,
    url: lastEntry.url,
    size: lastEntry.size,
  });
});
lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });

// Observe layout shifts (CLS)
let cumulativeLayoutShift = 0;
const clsObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const shift = entry as LayoutShift;
    if (!shift.hadRecentInput) { // Don't count user-initiated shifts
      cumulativeLayoutShift += shift.value;
    }
  }
});
clsObserver.observe({ type: 'layout-shift', buffered: true });
```

## web-vitals Library (Recommended)

The `web-vitals` library wraps the raw PerformanceObserver API with correct measurement semantics (including edge cases like back-forward cache navigations):

```bash
npm install web-vitals
```

```typescript
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

interface MetricPayload {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  id: string;            // unique per metric instance (useful for deduplication)
  navigationType: string; // 'navigate' | 'reload' | 'back-forward' | 'prerender'
  entries: PerformanceEntry[];
}

function sendToAnalytics(metric: MetricPayload): void {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    navigationType: metric.navigationType,
    url: window.location.href,
    userAgent: navigator.userAgent,
    connection: (navigator as any).connection?.effectiveType ?? 'unknown',
    timestamp: Date.now(),
  });

  // sendBeacon is fire-and-forget and works even during page unload
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/analytics/vitals', body);
  } else {
    // Fallback for unsupported browsers
    fetch('/analytics/vitals', {
      method: 'POST',
      body,
      headers: { 'Content-Type': 'application/json' },
      keepalive: true,
    });
  }
}

// Collect all Core Web Vitals
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

## Building a Custom RUM Dashboard

```typescript
// RUM data collection service
interface RUMEvent {
  sessionId: string;
  userId?: string;
  pageUrl: string;
  metric: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  device: {
    type: 'mobile' | 'tablet' | 'desktop';
    connection: string;
    memory?: number;
    cores?: number;
  };
  timing: number;
}

class RUMCollector {
  private sessionId: string;
  private buffer: RUMEvent[] = [];
  private flushTimer: ReturnType<typeof setTimeout> | null = null;

  constructor(private endpoint: string) {
    this.sessionId = this.generateSessionId();
    this.initCollectors();
    // Flush buffer on page hide
    window.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') this.flush(true);
    });
    window.addEventListener('pagehide', () => this.flush(true));
  }

  private generateSessionId(): string {
    return `${Date.now()}-${Math.random().toString(36).slice(2)}`;
  }

  private getDeviceInfo() {
    const nav = navigator as any;
    const width = window.innerWidth;

    return {
      type: width < 768 ? 'mobile' : width < 1024 ? 'tablet' : 'desktop',
      connection: nav.connection?.effectiveType ?? 'unknown',
      memory: nav.deviceMemory,
      cores: nav.hardwareConcurrency,
    } as RUMEvent['device'];
  }

  private record(metric: string, value: number, rating: MetricPayload['rating']): void {
    this.buffer.push({
      sessionId: this.sessionId,
      pageUrl: window.location.href,
      metric,
      value,
      rating,
      device: this.getDeviceInfo(),
      timing: Date.now(),
    });

    this.scheduleFlush();
  }

  private scheduleFlush(): void {
    if (this.flushTimer) return;
    this.flushTimer = setTimeout(() => {
      this.flushTimer = null;
      this.flush(false);
    }, 5000); // batch events, flush every 5s
  }

  private flush(immediate: boolean): void {
    if (this.buffer.length === 0) return;

    const events = [...this.buffer];
    this.buffer = [];

    const body = JSON.stringify({ events });

    if (immediate && navigator.sendBeacon) {
      navigator.sendBeacon(this.endpoint, body);
    } else {
      fetch(this.endpoint, {
        method: 'POST',
        body,
        headers: { 'Content-Type': 'application/json' },
        keepalive: immediate,
      }).catch(() => {
        // Re-add to buffer on failure
        this.buffer = events.concat(this.buffer);
      });
    }
  }

  private initCollectors(): void {
    onLCP((m) => this.record('LCP', m.value, m.rating));
    onINP((m) => this.record('INP', m.value, m.rating));
    onCLS((m) => this.record('CLS', m.value, m.rating));
    onFCP((m) => this.record('FCP', m.value, m.rating));
    onTTFB((m) => this.record('TTFB', m.value, m.rating));
  }
}
```

## Segmenting RUM Data

```typescript
// Raw metrics are only useful when segmented
// p75 across ALL users may hide that mobile on 3G is terrible

interface SegmentedMetrics {
  overall: PercentileStats;
  byDevice: Record<string, PercentileStats>;
  byConnection: Record<string, PercentileStats>;
  byPage: Record<string, PercentileStats>;
}

interface PercentileStats {
  p50: number;
  p75: number;
  p95: number;
  p99: number;
  sampleSize: number;
}

// SQL-like query on RUM data
const query = `
  SELECT
    percentile_cont(0.75) WITHIN GROUP (ORDER BY lcp_value) AS lcp_p75,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY lcp_value) AS lcp_p95,
    device_type,
    connection_type,
    COUNT(*) as n
  FROM rum_events
  WHERE metric = 'LCP'
    AND created_at > NOW() - INTERVAL '7 days'
    AND rating != 'unknown'
  GROUP BY device_type, connection_type
  ORDER BY lcp_p75 DESC
`;

// Common insights from segmentation:
// - Mobile p75 LCP = 4.2s (poor) while desktop p75 = 1.8s (good)
// - 3G users: 80% poor LCP; WiFi users: 15% poor LCP
// - /checkout page has p95 INP = 800ms (poor); other pages fine
```

## Privacy Considerations

```typescript
// RUM data must respect user privacy

// 1. No PII in metrics payloads
// ❌ Including user email in RUM event
{ metric: 'LCP', value: 2300, userId: 'alice@example.com' } // ← PII!

// ✅ Use anonymized identifiers
{ metric: 'LCP', value: 2300, userId: 'hash(alice@example.com)' } // hashed
// Or: session-level ID that doesn't tie back to a specific person

// 2. Respect consent (GDPR, CCPA)
// Only start RUM collection after consent is given
if (hasAnalyticsConsent()) {
  onLCP(sendToAnalytics);
  onINP(sendToAnalytics);
}

// 3. Do not track page URL parameters that may contain sensitive data
function sanitizeUrl(url: string): string {
  const u = new URL(url);
  // Remove sensitive query params
  ['token', 'key', 'secret', 'password', 'email'].forEach(p => u.searchParams.delete(p));
  return u.toString();
}

// 4. Sampling — don't send 100% of events if volume is high
const SAMPLE_RATE = 0.1; // 10% of sessions
if (Math.random() < SAMPLE_RATE) {
  initRUM();
}
```

## Commercial RUM Tools

```
Comparison of major RUM tools:

Tool              Open Core    Cost     Integration    Notable Feature
─────────────────────────────────────────────────────────────────────────
Datadog RUM       No           $$$$     1 script tag   Session replay
New Relic Browser No           $$$      1 script tag   APM correlation
Dynatrace         No           $$$$     Auto-inject    AI anomaly detect
Sentry Performance Partially  $$       SDK            Error + perf unified
Vercel Analytics  No           $$       Zero config    Edge-optimized
SpeedCurve        No           $$       Script tag     Competitive benchmarks
web-vitals + custom Free       Free     Custom         Full control, DIY
```

## Common Mistakes

### 1. Only Looking at Averages

```
p50 (median) hides that your worst 25% of users have terrible experiences.
Core Web Vitals are evaluated at the p75 in Google Search ranking.

❌ "Average LCP is 2.1s — we're good"
   Reality: p75 = 4.8s → failing for 25% of users → poor Google ranking

✅ Always look at p75 and p95 by segment (device, connection, page)
```

### 2. Not Sampling for High-Traffic Sites

```typescript
// ❌ Sending 100% of metrics on high-traffic site
// → Thousands of events per second → high cost, unnecessary

// ✅ Sample while still being statistically valid
const shouldSample = Math.random() < 0.05; // 5% sample → still millions of data points/day
if (shouldSample) {
  onLCP(sendToAnalytics);
  // weight each metric by 1/sampleRate for accurate aggregates
}
```

### 3. Ignoring Back-Forward Cache (bfcache)

```typescript
// ❌ Metrics from bfcache navigations look artificially fast
// When a user navigates back, the page is served from bfcache
// TTFB = 0ms, LCP = near instant — not representative of actual load

// ✅ web-vitals library handles this via navigationType
onLCP((metric) => {
  if (metric.navigationType === 'back-forward') {
    // Consider excluding or separately tracking bfcache navigations
    return;
  }
  sendToAnalytics(metric);
});
```

## Interview Questions

### 1. What is the difference between RUM and synthetic monitoring?

**Answer:** Synthetic monitoring runs scripted tests in controlled environments — fixed device emulation, throttled network, deterministic conditions. It's excellent for CI regression detection and pre-launch baselines because it's reproducible. RUM collects metrics from real users' actual devices, networks, and locations. A Lighthouse score of 95 on a fast MacBook doesn't mean users on a Galaxy A12 on 3G have a good experience. RUM reveals the actual distribution — p75 and p95 across device types and connections. Best practice is both: synthetic for regression detection in CI, RUM for understanding real-world impact.

### 2. What are Core Web Vitals and why do they matter beyond user experience?

**Answer:** Core Web Vitals are three browser-measured metrics Google uses in its search ranking algorithm: LCP (Largest Contentful Paint, loading), INP (Interaction to Next Paint, responsiveness), and CLS (Cumulative Layout Shift, visual stability). Good CWV scores (measured at p75 from field data) directly impact organic search ranking — pages with poor CWV are penalized relative to competitors. Beyond SEO, CWV are tightly correlated with conversion rate and user retention: Amazon found each 100ms improvement increased revenue by 1%; Google found a 0.1s improvement to mobile site speed increased conversions by 8%.

### 3. Why should you use navigator.sendBeacon instead of fetch for RUM data?

**Answer:** `sendBeacon` is designed for analytics-style reporting. It sends data asynchronously even when the page is being unloaded — when users navigate away or close the tab, a synchronous `fetch` would be cancelled, but `sendBeacon` queues the request in the browser process and delivers it even after the JavaScript context has been destroyed. `fetch` with `keepalive: true` is an alternative that works similarly but has a payload size limit. For page-unload metrics (final LCP, CLS score at end of session), `sendBeacon` ensures the data reaches your server reliably.

### 4. How do you handle privacy concerns when collecting RUM data?

**Answer:** Never include PII (names, emails, full user IDs) in RUM payloads — use hashed or anonymized identifiers. Respect consent mechanisms — don't initialize RUM collection until the user has consented (for GDPR/CCPA compliance). Sanitize URLs before sending — strip query parameters that may contain tokens, emails, or other sensitive data (a common source of accidental PII). Use sampling (5-10% of sessions) to reduce data volume, which also limits the personal data you collect. Store RUM data with a defined retention period (30-90 days is typical for performance data) and delete it afterward. Treat RUM data as personal data under GDPR — it can be tied to specific users via session IDs and behavioral patterns.
