# Application Performance Monitoring (APM)

## The Idea

**In plain English:** Application Performance Monitoring (APM) means continuously watching how fast and reliably your app runs in the real world — catching slowdowns, crashes, and bottlenecks before your users complain about them.

**Real-world analogy:** Think of a hospital's vital signs monitor attached to a patient — it tracks heart rate, blood pressure, and oxygen levels around the clock, and alarms go off the moment something dangerous happens.

- The patient = your running application
- The vital signs (heart rate, blood pressure) = metrics like response time, error rate, CPU usage
- The monitor screen = your APM dashboard
- The alarm = an alert firing when a metric crosses a threshold

---

## Overview

Application Performance Monitoring (APM) is the practice of measuring and tracking application performance in real-time to identify bottlenecks, optimize response times, and ensure excellent user experiences. APM tools provide visibility into application behavior, transaction traces, database queries, external API calls, and resource utilization to help teams proactively identify and resolve performance issues before they impact users.

## Table of Contents
- [APM Fundamentals](#apm-fundamentals)
- [Key Metrics](#key-metrics)
- [Transaction Tracing](#transaction-tracing)
- [Database Monitoring](#database-monitoring)
- [New Relic Integration](#new-relic-integration)
- [Datadog Integration](#datadog-integration)
- [Custom Instrumentation](#custom-instrumentation)
- [Performance Bottlenecks](#performance-bottlenecks)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## APM Fundamentals

### The Three Pillars of Observability

```
┌────────────────────────────────────────────────────────┐
│                   OBSERVABILITY                         │
├──────────────────┬──────────────────┬──────────────────┤
│     METRICS      │      LOGS        │      TRACES      │
├──────────────────┼──────────────────┼──────────────────┤
│ • Response time  │ • Error messages │ • Request flow   │
│ • Throughput     │ • State changes  │ • Service calls  │
│ • Error rate     │ • User actions   │ • Timing data    │
│ • Resource usage │ • Audit trail    │ • Dependencies   │
└──────────────────┴──────────────────┴──────────────────┘
```

### APM Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Frontend │  │ API      │  │ Worker   │  │ Database │  │
│  │          │  │ Server   │  │ Service  │  │          │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │             │              │              │         │
│       │   Instrumentation (APM Agent)            │         │
│       └─────────────┴──────────────┴──────────────┘         │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     APM Platform                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │  Data      │  │ Analysis   │  │ Alerting   │           │
│  │ Collection │→│ Engine     │→│ System     │           │
│  └────────────┘  └────────────┘  └────────────┘           │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │ Dashboard  │  │  Reports   │  │   APIs     │           │
│  │            │  │            │  │            │           │
│  └────────────┘  └────────────┘  └────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

## Key Metrics

### Golden Signals (SRE)

```javascript
// The Four Golden Signals
const goldenSignals = {
  // 1. Latency - Time to service a request
  latency: {
    p50: 45,      // 50th percentile (median)
    p95: 120,     // 95th percentile
    p99: 250,     // 99th percentile
    max: 1500,    // Maximum latency
  },
  
  // 2. Traffic - Demand on your system
  traffic: {
    requestsPerSecond: 1000,
    dailyActiveUsers: 50000,
    concurrentUsers: 500,
  },
  
  // 3. Errors - Rate of failed requests
  errors: {
    errorRate: 0.5,        // 0.5% error rate
    errorCount: 50,        // Errors per minute
    errorsByType: {
      '4xx': 30,
      '5xx': 20,
    },
  },
  
  // 4. Saturation - How full your service is
  saturation: {
    cpuUsage: 65,          // 65%
    memoryUsage: 78,       // 78%
    diskUsage: 45,         // 45%
    connectionPoolUsage: 80, // 80%
  },
};
```

### Apdex Score

```javascript
// Application Performance Index (Apdex)
// User satisfaction score from 0 to 1

class ApdexCalculator {
  constructor(threshold = 500) { // 500ms target
    this.threshold = threshold;
    this.toleranceThreshold = threshold * 4; // 2000ms
  }
  
  calculateApdex(responseTimes) {
    let satisfied = 0;
    let tolerating = 0;
    let frustrated = 0;
    
    responseTimes.forEach(time => {
      if (time <= this.threshold) {
        satisfied++;
      } else if (time <= this.toleranceThreshold) {
        tolerating++;
      } else {
        frustrated++;
      }
    });
    
    const total = responseTimes.length;
    const apdex = (satisfied + (tolerating / 2)) / total;
    
    return {
      score: apdex.toFixed(2),
      satisfied,
      tolerating,
      frustrated,
      rating: this.getApdexRating(apdex),
    };
  }
  
  getApdexRating(score) {
    if (score >= 0.94) return 'Excellent';
    if (score >= 0.85) return 'Good';
    if (score >= 0.70) return 'Fair';
    if (score >= 0.50) return 'Poor';
    return 'Unacceptable';
  }
}

// Usage
const apdex = new ApdexCalculator(500);
const responseTimes = [250, 450, 600, 1200, 300, 800, 2500];
const result = apdex.calculateApdex(responseTimes);

console.log(`Apdex Score: ${result.score} (${result.rating})`);
// Apdex Score: 0.71 (Fair)
```

### Custom Metrics

```typescript
// metrics-collector.ts
interface Metric {
  name: string;
  value: number;
  timestamp: number;
  tags?: Record<string, string>;
}

class MetricsCollector {
  private metrics: Metric[] = [];
  private counters: Map<string, number> = new Map();
  private histograms: Map<string, number[]> = new Map();
  
  // Counter: monotonically increasing value
  incrementCounter(name: string, value: number = 1, tags?: Record<string, string>): void {
    const current = this.counters.get(name) || 0;
    this.counters.set(name, current + value);
    
    this.record({
      name,
      value: current + value,
      timestamp: Date.now(),
      tags,
    });
  }
  
  // Gauge: point-in-time value
  recordGauge(name: string, value: number, tags?: Record<string, string>): void {
    this.record({
      name,
      value,
      timestamp: Date.now(),
      tags,
    });
  }
  
  // Histogram: distribution of values
  recordHistogram(name: string, value: number, tags?: Record<string, string>): void {
    const values = this.histograms.get(name) || [];
    values.push(value);
    this.histograms.set(name, values);
    
    this.record({
      name,
      value,
      timestamp: Date.now(),
      tags,
    });
  }
  
  // Timer: measure duration
  startTimer(name: string): () => void {
    const start = Date.now();
    return () => {
      const duration = Date.now() - start;
      this.recordHistogram(name, duration);
    };
  }
  
  private record(metric: Metric): void {
    this.metrics.push(metric);
  }
  
  getMetrics(): Metric[] {
    return this.metrics;
  }
  
  flush(): void {
    this.metrics = [];
  }
}

// Usage
const metrics = new MetricsCollector();

// Counter
metrics.incrementCounter('api.requests', 1, { endpoint: '/users', method: 'GET' });

// Gauge
metrics.recordGauge('memory.usage', process.memoryUsage().heapUsed, { service: 'api' });

// Histogram
metrics.recordHistogram('http.response.time', 145, { endpoint: '/users' });

// Timer
const endTimer = metrics.startTimer('database.query.duration');
await executeQuery();
endTimer();
```

## Transaction Tracing

### Distributed Tracing

```typescript
// trace-context.ts
import { v4 as uuidv4 } from 'uuid';

interface Span {
  spanId: string;
  traceId: string;
  parentSpanId?: string;
  name: string;
  startTime: number;
  endTime?: number;
  duration?: number;
  tags: Record<string, any>;
  logs: Array<{ timestamp: number; message: string }>;
}

class TraceContext {
  private spans: Map<string, Span> = new Map();
  private activeSpan: Span | null = null;
  
  startTrace(name: string): Span {
    const traceId = uuidv4();
    const spanId = uuidv4();
    
    const span: Span = {
      spanId,
      traceId,
      name,
      startTime: Date.now(),
      tags: {},
      logs: [],
    };
    
    this.spans.set(spanId, span);
    this.activeSpan = span;
    
    return span;
  }
  
  startSpan(name: string, parentSpan?: Span): Span {
    const parent = parentSpan || this.activeSpan;
    if (!parent) {
      throw new Error('No active trace');
    }
    
    const spanId = uuidv4();
    const span: Span = {
      spanId,
      traceId: parent.traceId,
      parentSpanId: parent.spanId,
      name,
      startTime: Date.now(),
      tags: {},
      logs: [],
    };
    
    this.spans.set(spanId, span);
    this.activeSpan = span;
    
    return span;
  }
  
  endSpan(span: Span): void {
    span.endTime = Date.now();
    span.duration = span.endTime - span.startTime;
    
    // Restore parent as active span
    if (span.parentSpanId) {
      this.activeSpan = this.spans.get(span.parentSpanId) || null;
    }
  }
  
  addTag(span: Span, key: string, value: any): void {
    span.tags[key] = value;
  }
  
  addLog(span: Span, message: string): void {
    span.logs.push({
      timestamp: Date.now(),
      message,
    });
  }
  
  getTrace(traceId: string): Span[] {
    return Array.from(this.spans.values())
      .filter(span => span.traceId === traceId);
  }
}

// Usage
const tracer = new TraceContext();

// Start trace
const rootSpan = tracer.startTrace('process-order');
tracer.addTag(rootSpan, 'orderId', '12345');

// Child span
const dbSpan = tracer.startSpan('database-query');
tracer.addTag(dbSpan, 'query', 'SELECT * FROM orders WHERE id = ?');
await fetchOrder();
tracer.endSpan(dbSpan);

// Another child span
const apiSpan = tracer.startSpan('external-api-call');
tracer.addTag(apiSpan, 'url', 'https://api.payment.com/charge');
await processPayment();
tracer.endSpan(apiSpan);

tracer.endSpan(rootSpan);

console.log(tracer.getTrace(rootSpan.traceId));
```

### Express Middleware for Tracing

```javascript
// express-tracing.js
const { v4: uuidv4 } = require('uuid');

function tracingMiddleware(req, res, next) {
  // Extract trace context from headers (W3C Trace Context)
  const traceParent = req.headers['traceparent'];
  const traceId = traceParent ? parseTraceParent(traceParent).traceId : uuidv4();
  const spanId = uuidv4();
  
  // Attach to request
  req.trace = {
    traceId,
    spanId,
    startTime: Date.now(),
  };
  
  // Set response headers
  res.setHeader('traceparent', `00-${traceId}-${spanId}-01`);
  
  // Track response
  const originalSend = res.send;
  res.send = function(data) {
    req.trace.endTime = Date.now();
    req.trace.duration = req.trace.endTime - req.trace.startTime;
    req.trace.statusCode = res.statusCode;
    
    // Send trace to APM
    sendTrace(req.trace, {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: req.trace.duration,
    });
    
    return originalSend.call(this, data);
  };
  
  next();
}

function parseTraceParent(traceparent) {
  const parts = traceparent.split('-');
  return {
    version: parts[0],
    traceId: parts[1],
    parentSpanId: parts[2],
    flags: parts[3],
  };
}

async function sendTrace(trace, metadata) {
  // Send to APM platform
  // Implementation depends on APM provider
}

module.exports = tracingMiddleware;
```

## Database Monitoring

### Query Performance Tracking

```javascript
// database-monitor.js
class DatabaseMonitor {
  constructor() {
    this.queries = [];
    this.slowQueryThreshold = 1000; // 1 second
  }
  
  trackQuery(query, params, duration) {
    const queryInfo = {
      query,
      params,
      duration,
      timestamp: Date.now(),
      isSlow: duration > this.slowQueryThreshold,
    };
    
    this.queries.push(queryInfo);
    
    // Alert on slow queries
    if (queryInfo.isSlow) {
      this.alertSlowQuery(queryInfo);
    }
    
    return queryInfo;
  }
  
  alertSlowQuery(queryInfo) {
    console.warn('[SLOW QUERY]', {
      query: queryInfo.query,
      duration: `${queryInfo.duration}ms`,
      threshold: `${this.slowQueryThreshold}ms`,
    });
  }
  
  getQueryStats() {
    const totalQueries = this.queries.length;
    const slowQueries = this.queries.filter(q => q.isSlow).length;
    const avgDuration = this.queries.reduce((sum, q) => sum + q.duration, 0) / totalQueries;
    
    const durations = this.queries.map(q => q.duration).sort((a, b) => a - b);
    const p95Index = Math.floor(durations.length * 0.95);
    const p99Index = Math.floor(durations.length * 0.99);
    
    return {
      totalQueries,
      slowQueries,
      slowQueryPercentage: (slowQueries / totalQueries * 100).toFixed(2),
      avgDuration: avgDuration.toFixed(2),
      p95: durations[p95Index],
      p99: durations[p99Index],
    };
  }
}

// Sequelize integration
const Sequelize = require('sequelize');
const dbMonitor = new DatabaseMonitor();

const sequelize = new Sequelize(DATABASE_URL, {
  logging: (sql, timing) => {
    dbMonitor.trackQuery(sql, [], timing);
  },
  benchmark: true, // Enable timing
});

// Mongoose integration
const mongoose = require('mongoose');

mongoose.set('debug', (collectionName, method, query, doc) => {
  const startTime = Date.now();
  
  mongoose.connection.db.collection(collectionName)[method](query, doc)
    .then(() => {
      const duration = Date.now() - startTime;
      dbMonitor.trackQuery(`${collectionName}.${method}`, [query, doc], duration);
    });
});
```

### Connection Pool Monitoring

```javascript
// pool-monitor.js
class ConnectionPoolMonitor {
  constructor(pool) {
    this.pool = pool;
    this.metrics = {
      acquireCount: 0,
      releaseCount: 0,
      timeoutCount: 0,
      waitTime: [],
    };
    
    this.startMonitoring();
  }
  
  startMonitoring() {
    setInterval(() => {
      this.collectMetrics();
    }, 60000); // Every minute
  }
  
  collectMetrics() {
    const stats = {
      timestamp: Date.now(),
      totalConnections: this.pool.totalCount,
      idleConnections: this.pool.idleCount,
      activeConnections: this.pool.totalCount - this.pool.idleCount,
      waitingClients: this.pool.waitingCount,
      maxConnections: this.pool.options.max,
      
      // Utilization
      utilizationPercent: ((this.pool.totalCount - this.pool.idleCount) / this.pool.options.max * 100).toFixed(2),
      
      // Cumulative metrics
      totalAcquires: this.metrics.acquireCount,
      totalReleases: this.metrics.releaseCount,
      totalTimeouts: this.metrics.timeoutCount,
      
      // Wait time stats
      avgWaitTime: this.calculateAvgWaitTime(),
    };
    
    // Alert on high utilization
    if (stats.utilizationPercent > 80) {
      console.warn('[POOL WARNING] High connection pool utilization:', stats);
    }
    
    // Send to APM
    this.sendToAPM(stats);
    
    return stats;
  }
  
  trackAcquire(waitTime) {
    this.metrics.acquireCount++;
    this.metrics.waitTime.push(waitTime);
  }
  
  trackRelease() {
    this.metrics.releaseCount++;
  }
  
  trackTimeout() {
    this.metrics.timeoutCount++;
  }
  
  calculateAvgWaitTime() {
    if (this.metrics.waitTime.length === 0) return 0;
    const sum = this.metrics.waitTime.reduce((a, b) => a + b, 0);
    return (sum / this.metrics.waitTime.length).toFixed(2);
  }
  
  sendToAPM(stats) {
    // Implementation depends on APM provider
  }
}

// PostgreSQL (pg) integration
const { Pool } = require('pg');
const pool = new Pool({ max: 20, connectionTimeoutMillis: 5000 });
const poolMonitor = new ConnectionPoolMonitor(pool);

// Wrap pool.connect to track metrics
const originalConnect = pool.connect.bind(pool);
pool.connect = async function() {
  const startTime = Date.now();
  
  try {
    const client = await originalConnect();
    const waitTime = Date.now() - startTime;
    poolMonitor.trackAcquire(waitTime);
    
    // Wrap release
    const originalRelease = client.release.bind(client);
    client.release = function() {
      poolMonitor.trackRelease();
      return originalRelease();
    };
    
    return client;
  } catch (error) {
    if (error.message.includes('timeout')) {
      poolMonitor.trackTimeout();
    }
    throw error;
  }
};
```

## New Relic Integration

### Node.js Setup

```javascript
// newrelic.js (must be first require in app)
'use strict';

exports.config = {
  app_name: ['My Application'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  
  // Logging
  logging: {
    level: 'info',
    filepath: 'stdout',
  },
  
  // Distributed tracing
  distributed_tracing: {
    enabled: true,
  },
  
  // Transaction tracer
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 'apdex_f',
    record_sql: 'obfuscated',
    explain_threshold: 500,
  },
  
  // Error collector
  error_collector: {
    enabled: true,
    ignore_status_codes: [404],
  },
  
  // Slow SQL
  slow_sql: {
    enabled: true,
  },
  
  // Custom attributes
  attributes: {
    include: [
      'request.parameters.*',
    ],
    exclude: [
      'request.headers.cookie',
      'request.headers.authorization',
    ],
  },
};

// app.js
require('newrelic'); // Must be first
const express = require('express');
const app = express();

// Custom instrumentation
const newrelic = require('newrelic');

app.get('/api/users/:id', async (req, res) => {
  // Start custom transaction
  const transaction = newrelic.getTransaction();
  
  // Add custom attributes
  newrelic.addCustomAttribute('userId', req.params.id);
  newrelic.addCustomAttribute('userRole', req.user?.role);
  
  try {
    // Create custom segment
    await newrelic.startSegment('fetchUserData', true, async () => {
      const user = await User.findById(req.params.id);
      return user;
    });
    
    // Record custom metric
    newrelic.recordMetric('Custom/UserFetch/Success', 1);
    
    res.json(user);
  } catch (error) {
    // Notice error
    newrelic.noticeError(error, {
      userId: req.params.id,
    });
    
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Custom instrumentation for third-party library
newrelic.instrument('my-library', function(shim, module) {
  shim.record(module, 'methodName', function(shim, fn, name, args) {
    return {
      name: 'MyLibrary/methodName',
      callback: shim.LAST,
    };
  });
});
```

### Browser Monitoring

```html
<!-- React/Angular: Inject New Relic Browser script -->
<!DOCTYPE html>
<html>
<head>
  <title>My App</title>
  <script type="text/javascript">
    window.NREUM || (NREUM = {}); NREUM.init = {
      distributed_tracing: { enabled: true },
      privacy: { cookies_enabled: true },
      ajax: { deny_list: ["bam.nr-data.net"] }
    };
    // New Relic Browser agent code (minified)
  </script>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

```typescript
// React: Custom page actions
import React, { useEffect } from 'react';

const UserDashboard: React.FC = () => {
  useEffect(() => {
    // Track page view
    if (window.newrelic) {
      window.newrelic.addPageAction('PageView', {
        page: 'UserDashboard',
        userId: currentUser.id,
      });
    }
  }, []);
  
  const handleExport = () => {
    // Track custom action
    if (window.newrelic) {
      window.newrelic.addPageAction('ExportData', {
        format: 'CSV',
        recordCount: data.length,
      });
    }
    
    exportToCsv(data);
  };
  
  return <div>Dashboard</div>;
};
```

## Datadog Integration

### Node.js Setup

```javascript
// tracer.js
const tracer = require('dd-trace').init({
  service: 'my-app',
  env: process.env.NODE_ENV,
  version: process.env.APP_VERSION,
  
  // APM
  analytics: true,
  runtimeMetrics: true,
  
  // Log injection
  logInjection: true,
  
  // Sampling
  sampleRate: 1.0, // 100% sampling
  
  // Tags
  tags: {
    team: 'backend',
    region: 'us-east-1',
  },
});

module.exports = tracer;

// app.js
require('./tracer'); // Must be first
const express = require('express');
const app = express();

// Datadog is auto-instrumented for Express

// Manual instrumentation
const tracer = require('dd-trace');

app.get('/api/users/:id', async (req, res) => {
  const span = tracer.scope().active();
  
  // Add tags
  span.setTag('user.id', req.params.id);
  span.setTag('resource.name', 'GET /api/users/:id');
  
  try {
    // Create child span
    const childSpan = tracer.startSpan('database.query', {
      childOf: span,
      tags: {
        'db.type': 'postgresql',
        'db.statement': 'SELECT * FROM users WHERE id = $1',
      },
    });
    
    const user = await User.findById(req.params.id);
    
    childSpan.finish();
    
    res.json(user);
  } catch (error) {
    // Add error to span
    span.setTag('error', true);
    span.setTag('error.message', error.message);
    span.setTag('error.stack', error.stack);
    
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Custom metrics
const StatsD = require('hot-shots');
const dogstatsd = new StatsD({
  host: 'localhost',
  port: 8125,
  prefix: 'myapp.',
  globalTags: ['env:production'],
});

// Send custom metrics
dogstatsd.increment('api.requests', 1, ['endpoint:/users', 'method:GET']);
dogstatsd.timing('api.response_time', 145, ['endpoint:/users']);
dogstatsd.gauge('queue.size', 42);
dogstatsd.histogram('payment.amount', 99.99, ['currency:USD']);
```

### Browser (RUM) Monitoring

```typescript
// datadog-rum.ts
import { datadogRum } from '@datadog/browser-rum';

datadogRum.init({
  applicationId: process.env.REACT_APP_DD_APPLICATION_ID || '',
  clientToken: process.env.REACT_APP_DD_CLIENT_TOKEN || '',
  site: 'datadoghq.com',
  service: 'my-frontend',
  env: process.env.NODE_ENV,
  version: process.env.REACT_APP_VERSION,
  
  // Session tracking
  sessionSampleRate: 100,
  sessionReplaySampleRate: 20,
  trackUserInteractions: true,
  trackResources: true,
  trackLongTasks: true,
  
  // Default privacy level
  defaultPrivacyLevel: 'mask-user-input',
});

datadogRum.startSessionReplayRecording();

// Set user context
export function setUser(user: { id: string; name: string; email: string }) {
  datadogRum.setUser({
    id: user.id,
    name: user.name,
    email: user.email,
  });
}

// Add custom action
export function trackAction(name: string, context?: Record<string, any>) {
  datadogRum.addAction(name, context);
}

// Add custom error
export function trackError(error: Error, context?: Record<string, any>) {
  datadogRum.addError(error, context);
}

// Track timing
export function trackTiming(name: string, duration: number) {
  datadogRum.addTiming(name, duration);
}

// Usage in React
import { trackAction, trackTiming } from './datadog-rum';

const CheckoutPage: React.FC = () => {
  const handleCheckout = async () => {
    const startTime = performance.now();
    
    trackAction('checkout_initiated', {
      cartValue: cart.total,
      itemCount: cart.items.length,
    });
    
    try {
      await processCheckout();
      
      const duration = performance.now() - startTime;
      trackTiming('checkout_duration', duration);
      
      trackAction('checkout_completed', {
        cartValue: cart.total,
        duration,
      });
    } catch (error) {
      trackError(error as Error, {
        step: 'payment',
        cartValue: cart.total,
      });
    }
  };
  
  return <button onClick={handleCheckout}>Checkout</button>;
};
```

## Custom Instrumentation

### Performance Timing API

```typescript
// performance-tracker.ts
class PerformanceTracker {
  private marks: Map<string, number> = new Map();
  
  mark(name: string): void {
    if (typeof performance !== 'undefined' && performance.mark) {
      performance.mark(name);
    }
    this.marks.set(name, Date.now());
  }
  
  measure(name: string, startMark: string, endMark?: string): number {
    if (typeof performance !== 'undefined' && performance.measure) {
      try {
        performance.measure(name, startMark, endMark);
        const measure = performance.getEntriesByName(name, 'measure')[0];
        return measure.duration;
      } catch (error) {
        // Fallback
      }
    }
    
    // Fallback calculation
    const startTime = this.marks.get(startMark);
    const endTime = endMark ? this.marks.get(endMark) : Date.now();
    
    if (startTime && endTime) {
      return endTime - startTime;
    }
    
    return 0;
  }
  
  getEntries(name?: string): PerformanceEntry[] {
    if (typeof performance !== 'undefined') {
      return name 
        ? performance.getEntriesByName(name)
        : performance.getEntries();
    }
    return [];
  }
  
  clearMarks(name?: string): void {
    if (typeof performance !== 'undefined') {
      if (name) {
        performance.clearMarks(name);
      } else {
        performance.clearMarks();
      }
    }
    
    if (name) {
      this.marks.delete(name);
    } else {
      this.marks.clear();
    }
  }
}

// Usage
const perf = new PerformanceTracker();

// Mark important points
perf.mark('api-call-start');
await fetchData();
perf.mark('api-call-end');

const duration = perf.measure('api-call', 'api-call-start', 'api-call-end');
console.log(`API call took ${duration}ms`);

// Web Vitals
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

getCLS(console.log); // Cumulative Layout Shift
getFID(console.log); // First Input Delay
getFCP(console.log); // First Contentful Paint
getLCP(console.log); // Largest Contentful Paint
getTTFB(console.log); // Time to First Byte
```

### Resource Monitoring

```javascript
// resource-monitor.js
class ResourceMonitor {
  constructor() {
    if (typeof performance !== 'undefined' && performance.getEntriesByType) {
      this.monitorResources();
    }
  }
  
  monitorResources() {
    // Observe new resources
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        this.analyzeResource(entry);
      });
    });
    
    observer.observe({ entryTypes: ['resource'] });
    
    // Process existing resources
    performance.getEntriesByType('resource').forEach((entry) => {
      this.analyzeResource(entry);
    });
  }
  
  analyzeResource(entry) {
    const resource = {
      name: entry.name,
      type: entry.initiatorType,
      duration: entry.duration,
      size: entry.transferSize,
      
      // Timing breakdown
      dns: entry.domainLookupEnd - entry.domainLookupStart,
      tcp: entry.connectEnd - entry.connectStart,
      ssl: entry.secureConnectionStart > 0 
        ? entry.connectEnd - entry.secureConnectionStart 
        : 0,
      ttfb: entry.responseStart - entry.requestStart,
      download: entry.responseEnd - entry.responseStart,
      
      // Cache hit
      cached: entry.transferSize === 0 && entry.decodedBodySize > 0,
    };
    
    // Flag slow resources
    if (resource.duration > 1000) {
      console.warn('[SLOW RESOURCE]', resource);
      this.sendToAPM(resource);
    }
    
    return resource;
  }
  
  sendToAPM(resource) {
    // Send to APM platform
  }
  
  getResourceStats() {
    const resources = performance.getEntriesByType('resource');
    
    const stats = {
      total: resources.length,
      byType: {},
      totalSize: 0,
      totalDuration: 0,
      cached: 0,
    };
    
    resources.forEach((entry) => {
      const type = entry.initiatorType;
      stats.byType[type] = (stats.byType[type] || 0) + 1;
      stats.totalSize += entry.transferSize || 0;
      stats.totalDuration += entry.duration;
      
      if (entry.transferSize === 0 && entry.decodedBodySize > 0) {
        stats.cached++;
      }
    });
    
    return stats;
  }
}

// Usage
const resourceMonitor = new ResourceMonitor();
```

## Performance Bottlenecks

### Identifying Bottlenecks

```javascript
// bottleneck-analyzer.js
class BottleneckAnalyzer {
  constructor() {
    this.transactions = [];
  }
  
  analyzeTransaction(transaction) {
    const bottlenecks = [];
    
    // Database queries
    const dbQueries = transaction.spans.filter(s => s.type === 'db');
    const totalDbTime = dbQueries.reduce((sum, s) => sum + s.duration, 0);
    
    if (totalDbTime > transaction.duration * 0.5) {
      bottlenecks.push({
        type: 'database',
        severity: 'high',
        message: 'Database queries account for >50% of transaction time',
        impact: totalDbTime,
        suggestions: [
          'Add database indexes',
          'Optimize queries',
          'Implement caching',
          'Use database connection pooling',
        ],
      });
    }
    
    // N+1 queries
    const sequentialQueries = this.detectNPlusOne(dbQueries);
    if (sequentialQueries.length > 5) {
      bottlenecks.push({
        type: 'n-plus-one',
        severity: 'critical',
        message: `Detected ${sequentialQueries.length} sequential queries (N+1 problem)`,
        impact: sequentialQueries.reduce((sum, s) => sum + s.duration, 0),
        suggestions: [
          'Use eager loading/joins',
          'Implement data loader pattern',
          'Batch queries',
        ],
      });
    }
    
    // External API calls
    const apiCalls = transaction.spans.filter(s => s.type === 'http');
    const totalApiTime = apiCalls.reduce((sum, s) => sum + s.duration, 0);
    
    if (totalApiTime > transaction.duration * 0.3) {
      bottlenecks.push({
        type: 'external-api',
        severity: 'medium',
        message: 'External API calls are slow',
        impact: totalApiTime,
        suggestions: [
          'Cache API responses',
          'Use faster API endpoints',
          'Implement timeout and retry logic',
          'Consider CDN for static resources',
        ],
      });
    }
    
    // Large payloads
    const largePayloads = transaction.spans.filter(s => s.size > 1000000); // 1MB
    if (largePayloads.length > 0) {
      bottlenecks.push({
        type: 'large-payload',
        severity: 'medium',
        message: 'Large response payloads detected',
        impact: largePayloads.reduce((sum, s) => sum + s.size, 0),
        suggestions: [
          'Implement pagination',
          'Use compression (gzip)',
          'Return only necessary fields',
          'Implement streaming for large data',
        ],
      });
    }
    
    return bottlenecks;
  }
  
  detectNPlusOne(queries) {
    const sequential = [];
    
    for (let i = 1; i < queries.length; i++) {
      const prev = queries[i - 1];
      const curr = queries[i];
      
      // Check if queries are sequential (not overlapping)
      if (prev.endTime <= curr.startTime) {
        // Check if similar queries (same table/collection)
        if (this.areSimilarQueries(prev.query, curr.query)) {
          sequential.push(curr);
        }
      }
    }
    
    return sequential;
  }
  
  areSimilarQueries(query1, query2) {
    // Simple heuristic: same table/collection name
    const table1 = this.extractTableName(query1);
    const table2 = this.extractTableName(query2);
    return table1 === table2;
  }
  
  extractTableName(query) {
    // Simplified: extract FROM clause
    const match = query.match(/FROM\s+(\w+)/i);
    return match ? match[1] : '';
  }
}
```

### Memory Leak Detection

```javascript
// memory-monitor.js
class MemoryMonitor {
  constructor() {
    this.snapshots = [];
    this.heapGrowthThreshold = 50 * 1024 * 1024; // 50MB
    this.startMonitoring();
  }
  
  startMonitoring() {
    setInterval(() => {
      this.takeSnapshot();
    }, 60000); // Every minute
  }
  
  takeSnapshot() {
    if (typeof process === 'undefined') {
      return; // Not in Node.js
    }
    
    const usage = process.memoryUsage();
    
    const snapshot = {
      timestamp: Date.now(),
      heapUsed: usage.heapUsed,
      heapTotal: usage.heapTotal,
      external: usage.external,
      rss: usage.rss,
    };
    
    this.snapshots.push(snapshot);
    
    // Keep last 60 snapshots (1 hour)
    if (this.snapshots.length > 60) {
      this.snapshots.shift();
    }
    
    this.detectLeaks();
  }
  
  detectLeaks() {
    if (this.snapshots.length < 10) {
      return; // Not enough data
    }
    
    const recent = this.snapshots.slice(-10);
    const oldest = recent[0];
    const newest = recent[recent.length - 1];
    
    const heapGrowth = newest.heapUsed - oldest.heapUsed;
    const timeWindow = (newest.timestamp - oldest.timestamp) / 1000 / 60; // minutes
    
    if (heapGrowth > this.heapGrowthThreshold) {
      const growthRate = heapGrowth / timeWindow; // bytes per minute
      
      console.warn('[MEMORY LEAK WARNING]', {
        heapGrowth: `${(heapGrowth / 1024 / 1024).toFixed(2)} MB`,
        timeWindow: `${timeWindow.toFixed(2)} minutes`,
        growthRate: `${(growthRate / 1024).toFixed(2)} KB/min`,
        currentHeap: `${(newest.heapUsed / 1024 / 1024).toFixed(2)} MB`,
      });
      
      // Trigger heap dump for analysis
      this.createHeapDump();
    }
  }
  
  createHeapDump() {
    if (typeof process === 'undefined') return;
    
    const heapdump = require('heapdump');
    const filename = `heapdump-${Date.now()}.heapsnapshot`;
    
    heapdump.writeSnapshot(filename, (err) => {
      if (err) {
        console.error('Failed to create heap dump:', err);
      } else {
        console.log('Heap dump created:', filename);
      }
    });
  }
  
  getMemoryStats() {
    if (this.snapshots.length === 0) {
      return null;
    }
    
    const latest = this.snapshots[this.snapshots.length - 1];
    const oldest = this.snapshots[0];
    
    return {
      current: {
        heapUsed: `${(latest.heapUsed / 1024 / 1024).toFixed(2)} MB`,
        heapTotal: `${(latest.heapTotal / 1024 / 1024).toFixed(2)} MB`,
        rss: `${(latest.rss / 1024 / 1024).toFixed(2)} MB`,
      },
      trend: {
        heapGrowth: `${((latest.heapUsed - oldest.heapUsed) / 1024 / 1024).toFixed(2)} MB`,
        timeWindow: `${((latest.timestamp - oldest.timestamp) / 1000 / 60).toFixed(2)} minutes`,
      },
    };
  }
}

// Usage
const memoryMonitor = new MemoryMonitor();
```

## Common Mistakes

### 1. Not Setting Baseline Performance

```javascript
// Bad: No baseline, can't detect regressions
app.get('/api/users', async (req, res) => {
  const users = await fetchUsers();
  res.json(users);
});

// Good: Establish and monitor baseline
const performanceBaseline = {
  '/api/users': { p95: 200, p99: 500 }, // ms
  '/api/orders': { p95: 300, p99: 800 },
};

app.get('/api/users', async (req, res) => {
  const start = Date.now();
  const users = await fetchUsers();
  const duration = Date.now() - start;
  
  // Alert if regression detected
  if (duration > performanceBaseline['/api/users'].p95 * 1.5) {
    alertPerformanceRegression('/api/users', duration);
  }
  
  res.json(users);
});
```

### 2. Over-Instrumentation

```javascript
// Bad: Too much instrumentation overhead
function processData(items) {
  items.forEach(item => {
    tracer.startSpan(`process-item-${item.id}`); // Too granular!
    processItem(item);
    tracer.endSpan();
  });
}

// Good: Instrument at appropriate level
function processData(items) {
  const span = tracer.startSpan('process-data');
  span.setTag('item_count', items.length);
  
  items.forEach(item => processItem(item));
  
  span.finish();
}
```

### 3. Ignoring External Dependencies

```javascript
// Bad: Only monitoring internal code
async function getUser(id) {
  return await User.findById(id); // No monitoring
}

// Good: Monitor external dependencies
async function getUser(id) {
  const span = tracer.startSpan('get-user');
  span.setTag('user.id', id);
  
  // Monitor database call
  const dbSpan = tracer.startSpan('database.query', { childOf: span });
  dbSpan.setTag('db.statement', 'SELECT * FROM users WHERE id = ?');
  
  try {
    const user = await User.findById(id);
    dbSpan.setTag('db.rows_affected', user ? 1 : 0);
    return user;
  } finally {
    dbSpan.finish();
    span.finish();
  }
}
```

### 4. Not Monitoring in Development

```javascript
// Bad: Only monitoring in production
if (process.env.NODE_ENV === 'production') {
  require('./apm');
}

// Good: Monitor in all environments (with sampling)
require('./apm').init({
  service: 'my-app',
  environment: process.env.NODE_ENV,
  sampleRate: process.env.NODE_ENV === 'production' ? 1.0 : 0.1,
});
```

### 5. Missing Context in Traces

```javascript
// Bad: Generic traces without context
tracer.startSpan('api-call');

// Good: Rich context
tracer.startSpan('api-call', {
  tags: {
    'http.method': 'POST',
    'http.url': '/api/orders',
    'user.id': userId,
    'order.total': total,
    'payment.method': paymentMethod,
  },
});
```

## Best Practices

### 1. Define SLOs (Service Level Objectives)

```javascript
// Define SLOs for your service
const SLOs = {
  availability: {
    target: 99.9, // 99.9% uptime
    measurement: 'uptime_percentage',
  },
  latency: {
    target: 200, // p95 < 200ms
    measurement: 'p95_response_time',
  },
  errorRate: {
    target: 0.1, // <0.1% error rate
    measurement: 'error_percentage',
  },
};

// Monitor SLO compliance
function checkSLOCompliance(metrics) {
  const violations = [];
  
  if (metrics.uptimePercentage < SLOs.availability.target) {
    violations.push({
      slo: 'availability',
      actual: metrics.uptimePercentage,
      target: SLOs.availability.target,
    });
  }
  
  if (metrics.p95ResponseTime > SLOs.latency.target) {
    violations.push({
      slo: 'latency',
      actual: metrics.p95ResponseTime,
      target: SLOs.latency.target,
    });
  }
  
  if (metrics.errorPercentage > SLOs.errorRate.target) {
    violations.push({
      slo: 'errorRate',
      actual: metrics.errorPercentage,
      target: SLOs.errorRate.target,
    });
  }
  
  if (violations.length > 0) {
    alertSLOViolation(violations);
  }
  
  return violations;
}
```

### 2. Implement Synthetic Monitoring

```javascript
// synthetic-monitor.js
const puppeteer = require('puppeteer');

class SyntheticMonitor {
  async runUserJourney() {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    
    const metrics = {
      steps: [],
      totalDuration: 0,
      errors: [],
    };
    
    try {
      // Step 1: Home page
      const homeStart = Date.now();
      await page.goto('https://example.com');
      metrics.steps.push({
        name: 'home-page-load',
        duration: Date.now() - homeStart,
      });
      
      // Step 2: Login
      const loginStart = Date.now();
      await page.type('#email', 'test@example.com');
      await page.type('#password', 'password');
      await page.click('#login-button');
      await page.waitForNavigation();
      metrics.steps.push({
        name: 'login',
        duration: Date.now() - loginStart,
      });
      
      // Step 3: Navigate to dashboard
      const dashboardStart = Date.now();
      await page.goto('https://example.com/dashboard');
      await page.waitForSelector('.dashboard');
      metrics.steps.push({
        name: 'dashboard-load',
        duration: Date.now() - dashboardStart,
      });
      
      metrics.totalDuration = metrics.steps.reduce((sum, step) => sum + step.duration, 0);
      
    } catch (error) {
      metrics.errors.push({
        step: metrics.steps.length,
        error: error.message,
      });
    } finally {
      await browser.close();
    }
    
    // Send metrics to APM
    this.sendMetrics(metrics);
    
    return metrics;
  }
  
  sendMetrics(metrics) {
    // Implementation depends on APM provider
  }
}

// Run every 5 minutes
const monitor = new SyntheticMonitor();
setInterval(() => monitor.runUserJourney(), 5 * 60 * 1000);
```

### 3. Create Performance Budgets

```javascript
// performance-budget.js
const performanceBudgets = {
  // Page load metrics
  'time-to-first-byte': 200,
  'first-contentful-paint': 1500,
  'largest-contentful-paint': 2500,
  'time-to-interactive': 3500,
  
  // Resource sizes
  'total-page-size': 2000 * 1024, // 2MB
  'javascript-size': 500 * 1024,  // 500KB
  'css-size': 200 * 1024,         // 200KB
  'image-size': 1000 * 1024,      // 1MB
  
  // API response times
  'api-response-time': 500,        // 500ms
};

function checkPerformanceBudget(metrics) {
  const violations = [];
  
  Object.keys(performanceBudgets).forEach(metric => {
    if (metrics[metric] > performanceBudgets[metric]) {
      violations.push({
        metric,
        actual: metrics[metric],
        budget: performanceBudgets[metric],
        overBy: metrics[metric] - performanceBudgets[metric],
      });
    }
  });
  
  if (violations.length > 0) {
    console.error('Performance budget violations:', violations);
    // Fail CI build if in CI environment
    if (process.env.CI) {
      process.exit(1);
    }
  }
  
  return violations;
}
```

### 4. Implement Alerting Tiers

```javascript
// alerting-tiers.js
const AlertSeverity = {
  INFO: { level: 1, channel: 'slack' },
  WARNING: { level: 2, channel: 'slack' },
  ERROR: { level: 3, channel: ['slack', 'email'] },
  CRITICAL: { level: 4, channel: ['slack', 'email', 'pagerduty'] },
};

function sendAlert(severity, message, context) {
  const config = AlertSeverity[severity];
  
  const channels = Array.isArray(config.channel) ? config.channel : [config.channel];
  
  channels.forEach(channel => {
    switch (channel) {
      case 'slack':
        sendSlackAlert(message, context);
        break;
      case 'email':
        sendEmailAlert(message, context);
        break;
      case 'pagerduty':
        sendPagerDutyAlert(message, context);
        break;
    }
  });
}

// Usage
if (errorRate > 5) {
  sendAlert('CRITICAL', 'High error rate detected', {
    errorRate,
    threshold: 5,
    service: 'api',
  });
} else if (errorRate > 1) {
  sendAlert('WARNING', 'Elevated error rate', {
    errorRate,
    threshold: 1,
  });
}
```

### 5. Regular Performance Reviews

```javascript
// Generate weekly performance report
async function generatePerformanceReport(startDate, endDate) {
  const metrics = await fetchMetrics(startDate, endDate);
  
  const report = {
    period: { start: startDate, end: endDate },
    
    // Overall health
    health: {
      availability: metrics.uptime,
      errorRate: metrics.errorRate,
      p95Latency: metrics.p95,
    },
    
    // Trends
    trends: {
      latencyTrend: calculateTrend(metrics.latencyOverTime),
      errorTrend: calculateTrend(metrics.errorsOverTime),
      throughputTrend: calculateTrend(metrics.throughputOverTime),
    },
    
    // Top issues
    topIssues: [
      ...metrics.slowestEndpoints.slice(0, 5),
      ...metrics.mostFrequentErrors.slice(0, 5),
    ],
    
    // Recommendations
    recommendations: generateRecommendations(metrics),
  };
  
  // Send report
  await sendReport(report);
  
  return report;
}

function generateRecommendations(metrics) {
  const recommendations = [];
  
  if (metrics.p95 > 1000) {
    recommendations.push('Consider caching to improve response times');
  }
  
  if (metrics.databaseQueryTime > metrics.totalTime * 0.5) {
    recommendations.push('Database queries are a bottleneck - add indexes or optimize queries');
  }
  
  if (metrics.externalApiTime > metrics.totalTime * 0.3) {
    recommendations.push('External API calls are slow - implement caching or find faster alternatives');
  }
  
  return recommendations;
}

// Schedule weekly report
schedule.scheduleJob('0 9 * * MON', () => {
  const endDate = new Date();
  const startDate = new Date(endDate.getTime() - 7 * 24 * 60 * 60 * 1000);
  generatePerformanceReport(startDate, endDate);
});
```

## When to Use/Not to Use

### When to Use APM

1. **Production Applications**: Monitor real user experience
2. **Performance-Critical Systems**: Low latency requirements
3. **Distributed Systems**: Track requests across services
4. **High Traffic Applications**: Identify bottlenecks at scale
5. **SLA Commitments**: Need to meet service level agreements
6. **Microservices**: Complex inter-service dependencies
7. **Troubleshooting**: Difficult to reproduce issues
8. **Capacity Planning**: Understand resource utilization

### When APM May Be Overkill

1. **Simple Applications**: Static websites, minimal logic
2. **Internal Tools**: Low traffic, few users
3. **Prototypes**: Early development, frequent changes
4. **Budget Constraints**: APM tools can be expensive
5. **Single-Page Static Sites**: No backend, minimal JS
6. **Scheduled Jobs**: Simple cron tasks
7. **Legacy Systems**: No budget for instrumentation
8. **Development Environment**: Use profilers instead

## Interview Questions

### 1. What are the four golden signals of monitoring?

**Answer**: The four golden signals from Google's SRE book are: 1) Latency (time to serve requests), 2) Traffic (demand on the system), 3) Errors (rate of failed requests), 4) Saturation (how full the service is - CPU, memory, I/O). These provide comprehensive health indicators for any service.

### 2. What is distributed tracing and why is it important?

**Answer**: Distributed tracing tracks requests as they flow through multiple services in a distributed system. Each service adds a span to the trace, creating a complete picture of the request journey. Important for: identifying bottlenecks across services, understanding service dependencies, debugging performance issues, and optimizing end-to-end latency in microservices architectures.

### 3. How do you identify performance bottlenecks in an application?

**Answer**: Use APM tools to analyze: 1) Transaction traces to identify slow operations, 2) Database query performance (slow queries, N+1 problems), 3) External API call latency, 4) Resource utilization (CPU, memory, I/O), 5) Network latency, 6) Code-level profiling for hot paths. Look for operations taking >30% of transaction time as primary bottlenecks.

### 4. What is the difference between metrics, logs, and traces?

**Answer**: Metrics are aggregated numerical data over time (request rate, latency percentiles). Logs are timestamped discrete events with detailed context. Traces show the path of a single request through the system. Use metrics for alerting, logs for debugging, and traces for understanding system behavior. Together they provide complete observability.

### 5. How would you set performance SLOs for a web application?

**Answer**: Define SLOs based on user expectations and business requirements: 1) Availability SLO (e.g., 99.9% uptime), 2) Latency SLO (e.g., p95 < 200ms), 3) Error rate SLO (e.g., <0.1% errors). Consider user-facing vs internal APIs. Set realistic targets based on historical data, monitor compliance, and establish error budgets for acceptable downtime.

### 6. What is the N+1 query problem and how do you detect it?

**Answer**: N+1 problem occurs when code executes 1 query to fetch N items, then N additional queries to fetch related data for each item (total N+1 queries). Detect by: monitoring sequential database queries in APM, looking for patterns of similar queries, checking transaction traces for many short database operations. Fix with eager loading, joins, or dataloader pattern to batch queries.

### 7. How do you monitor client-side performance?

**Answer**: Use Real User Monitoring (RUM) to track: Core Web Vitals (LCP, FID, CLS), navigation timing (TTFB, DOM load, onload), resource loading times, JavaScript errors, and user interactions. Tools like Datadog RUM, New Relic Browser, or Google Analytics provide client-side instrumentation. Monitor by geography, device, and browser to identify specific issues.

### 8. What is Apdex and how is it calculated?

**Answer**: Apdex (Application Performance Index) measures user satisfaction on a 0-1 scale. Set a target response time (e.g., 500ms). Requests ≤target are satisfied, target-4×target are tolerating, >4×target are frustrated. Apdex = (satisfied + tolerating/2) / total. Score >0.94 is excellent, 0.85-0.94 is good, 0.70-0.85 is fair, 0.50-0.70 is poor, <0.50 is unacceptable.

## Key Takeaways

1. **Monitor the four golden signals**: Latency, traffic, errors, saturation provide comprehensive visibility
2. **Use distributed tracing**: Essential for understanding microservices performance
3. **Set performance baselines**: Establish SLOs and monitor regressions
4. **Identify bottlenecks systematically**: Database, external APIs, code inefficiencies
5. **Implement appropriate instrumentation**: Balance detail with overhead
6. **Monitor both server and client**: Full-stack observability
7. **Create actionable alerts**: Tier by severity, avoid alert fatigue
8. **Regular performance reviews**: Identify trends and areas for improvement
9. **Performance budgets**: Prevent regressions in CI/CD
10. **Correlate metrics, logs, and traces**: Unified observability for faster debugging

## Resources

### APM Platforms
- [New Relic](https://newrelic.com/) - Full-stack observability
- [Datadog](https://www.datadoghq.com/) - Monitoring and analytics
- [AppDynamics](https://www.appdynamics.com/) - Application performance
- [Dynatrace](https://www.dynatrace.com/) - Software intelligence
- [Elastic APM](https://www.elastic.co/apm/) - Open-source APM

### Open Source Tools
- [Prometheus](https://prometheus.io/) - Metrics and alerting
- [Grafana](https://grafana.com/) - Visualization
- [Jaeger](https://www.jaegertracing.io/) - Distributed tracing
- [Zipkin](https://zipkin.io/) - Distributed tracing
- [OpenTelemetry](https://opentelemetry.io/) - Observability framework

### Documentation
- [Google SRE Book](https://sre.google/books/)
- [OpenTelemetry Docs](https://opentelemetry.io/docs/)
- [Web Vitals](https://web.dev/vitals/)

### Articles
- [The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Distributed Tracing](https://microservices.io/patterns/observability/distributed-tracing.html)
- [Performance Budgets](https://web.dev/performance-budgets-101/)
