# Performance Budgets

## Overview

Performance budgets are constraints that define acceptable performance thresholds for your application. They prevent performance regressions by enforcing limits on metrics (timing, file sizes, request counts) in development and CI/CD. This guide covers budget setting strategies, CI enforcement with Lighthouse CI, bundle size tracking, webpack-bundle-analyzer usage, and automated performance governance.

## What Are Performance Budgets

Performance budgets are thresholds for performance metrics:

```
Types of Performance Budgets:

1. Metric Budgets (Timing):
┌────────────────────────────────────────────────┐
│  Metric               Budget      Current      │
├────────────────────────────────────────────────┤
│  FCP                  < 1.8s       1.2s  ✓    │
│  LCP                  < 2.5s       2.8s  ✗    │
│  TBT                  < 200ms      150ms ✓    │
│  CLS                  < 0.1        0.05  ✓    │
└────────────────────────────────────────────────┘

2. Resource Budgets (Size):
┌────────────────────────────────────────────────┐
│  Resource Type        Budget      Current      │
├────────────────────────────────────────────────┤
│  JavaScript           < 200KB      180KB ✓    │
│  CSS                  < 50KB       45KB  ✓    │
│  Images               < 500KB      520KB ✗    │
│  Fonts                < 100KB      80KB  ✓    │
│  Total                < 1MB        825KB ✓    │
└────────────────────────────────────────────────┘

3. Request Budgets (Count):
┌────────────────────────────────────────────────┐
│  Resource Type        Budget      Current      │
├────────────────────────────────────────────────┤
│  Total Requests       < 50         42    ✓    │
│  JS Requests          < 10         8     ✓    │
│  CSS Requests         < 5          3     ✓    │
│  Image Requests       < 20         18    ✓    │
└────────────────────────────────────────────────┘
```

## Setting Performance Budgets

### Research-Based Approach

```typescript
// 1. Analyze your current performance
interface PerformanceBaseline {
  p50: number; // 50th percentile
  p75: number; // 75th percentile
  p90: number; // 90th percentile
  p95: number; // 95th percentile
}

const currentMetrics: Record<string, PerformanceBaseline> = {
  fcp: { p50: 1200, p75: 1800, p90: 2400, p95: 3000 },
  lcp: { p50: 2000, p75: 2800, p90: 3500, p95: 4200 },
  tbt: { p50: 100, p75: 200, p90: 350, p95: 500 }
};

// 2. Set budgets based on goals
// Goal: 75% of users have "Good" experience
const budgets = {
  fcp: 1800,  // Good threshold from Core Web Vitals
  lcp: 2500,  // Good threshold
  tbt: 200,   // Good threshold
  cls: 0.1    // Good threshold
};

// 3. Set resource budgets based on performance requirements
const resourceBudgets = {
  // 3G connection: ~400KB/s download
  // Target: < 5s load time on 3G
  // Formula: (5s * 400KB/s) = 2MB total budget
  // After compression (~70%), aim for ~600KB gzipped
  
  javascript: {
    total: 200, // KB gzipped
    initial: 100, // KB for initial bundle
    perRoute: 50 // KB per lazy-loaded route
  },
  
  css: {
    total: 50,
    critical: 15
  },
  
  images: {
    total: 500,
    hero: 100,
    perImage: 50
  },
  
  fonts: {
    total: 100,
    perFont: 30
  }
};
```

### Industry Benchmarks

```typescript
// Based on HTTP Archive and Core Web Vitals thresholds

const industryBenchmarks = {
  // Top 10% of sites (aggressive budgets)
  aggressive: {
    fcp: 1000,
    lcp: 1500,
    tbt: 150,
    cls: 0.05,
    totalJs: 150,  // KB
    totalCss: 30,
    totalImages: 300
  },
  
  // Top 25% of sites (recommended budgets)
  recommended: {
    fcp: 1800,
    lcp: 2500,
    tbt: 200,
    cls: 0.1,
    totalJs: 200,
    totalCss: 50,
    totalImages: 500
  },
  
  // Top 50% of sites (minimum acceptable)
  minimum: {
    fcp: 2400,
    lcp: 4000,
    tbt: 300,
    cls: 0.25,
    totalJs: 300,
    totalCss: 75,
    totalImages: 800
  }
};
```

## Lighthouse CI Integration

Lighthouse CI enforces performance budgets in CI/CD:

### Installation and Setup

```bash
# Install Lighthouse CI
npm install --save-dev @lhci/cli

# Initialize
lhci wizard
```

### Configuration with Budgets

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "url": [
        "http://localhost:3000",
        "http://localhost:3000/about",
        "http://localhost:3000/products"
      ],
      "numberOfRuns": 3,
      "settings": {
        "preset": "desktop",
        "throttlingMethod": "simulate",
        "throttling": {
          "cpuSlowdownMultiplier": 1
        }
      }
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        // Metric budgets
        "first-contentful-paint": ["error", {"maxNumericValue": 1800}],
        "largest-contentful-paint": ["error", {"maxNumericValue": 2500}],
        "cumulative-layout-shift": ["error", {"maxNumericValue": 0.1}],
        "total-blocking-time": ["error", {"maxNumericValue": 200}],
        "speed-index": ["error", {"maxNumericValue": 3000}],
        
        // Performance score
        "categories:performance": ["error", {"minScore": 0.9}],
        
        // Resource budgets
        "resource-summary:script:size": ["error", {"maxNumericValue": 204800}],
        "resource-summary:stylesheet:size": ["error", {"maxNumericValue": 51200}],
        "resource-summary:image:size": ["error", {"maxNumericValue": 512000}],
        "resource-summary:font:size": ["error", {"maxNumericValue": 102400}],
        "resource-summary:total:size": ["error", {"maxNumericValue": 1048576}],
        
        // Request count budgets
        "resource-summary:script:count": ["warn", {"maxNumericValue": 10}],
        "resource-summary:stylesheet:count": ["warn", {"maxNumericValue": 5}],
        "resource-summary:third-party:count": ["warn", {"maxNumericValue": 10}]
      }
    },
    "upload": {
      "target": "temporary-public-storage"
    }
  }
}
```

### GitHub Actions Workflow

```yaml
# .github/workflows/performance-budget.yml
name: Performance Budget

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  performance:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Serve
        run: |
          npm install -g serve
          serve -s build -l 3000 &
          sleep 5
      
      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
      
      - name: Upload Results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: lighthouse-results
          path: .lighthouseci
```

### Custom Budget Checker

```typescript
// budget-checker.ts
import { readFileSync } from 'fs';
import { join } from 'path';

interface Budget {
  metric: string;
  budget: number;
  actual: number;
  passed: boolean;
}

interface BudgetConfig {
  metrics: Record<string, number>;
  resources: Record<string, number>;
}

class BudgetChecker {
  private config: BudgetConfig;
  private results: Budget[] = [];
  
  constructor(configPath: string) {
    this.config = JSON.parse(readFileSync(configPath, 'utf-8'));
  }
  
  async check(lighthouseReport: any): Promise<Budget[]> {
    // Check metric budgets
    for (const [metric, budget] of Object.entries(this.config.metrics)) {
      const actual = this.getMetricValue(lighthouseReport, metric);
      
      this.results.push({
        metric,
        budget,
        actual,
        passed: actual <= budget
      });
    }
    
    // Check resource budgets
    const resourceSummary = lighthouseReport.audits['resource-summary'];
    
    for (const [resourceType, budget] of Object.entries(this.config.resources)) {
      const actual = this.getResourceSize(resourceSummary, resourceType);
      
      this.results.push({
        metric: `${resourceType}-size`,
        budget,
        actual,
        passed: actual <= budget
      });
    }
    
    return this.results;
  }
  
  private getMetricValue(report: any, metric: string): number {
    const auditMap: Record<string, string> = {
      fcp: 'first-contentful-paint',
      lcp: 'largest-contentful-paint',
      tbt: 'total-blocking-time',
      cls: 'cumulative-layout-shift'
    };
    
    const auditId = auditMap[metric];
    return report.audits[auditId]?.numericValue || 0;
  }
  
  private getResourceSize(resourceSummary: any, type: string): number {
    const items = resourceSummary.details?.items || [];
    const item = items.find((i: any) => i.resourceType === type);
    return item?.transferSize || 0;
  }
  
  printResults(): void {
    console.log('\n=== Performance Budget Results ===\n');
    
    let passed = 0;
    let failed = 0;
    
    this.results.forEach(result => {
      const status = result.passed ? '✓ PASS' : '✗ FAIL';
      const color = result.passed ? '\x1b[32m' : '\x1b[31m';
      
      console.log(
        `${color}${status}\x1b[0m ${result.metric}: ` +
        `${result.actual.toFixed(2)} / ${result.budget} ` +
        `(${((result.actual / result.budget) * 100).toFixed(1)}%)`
      );
      
      if (result.passed) passed++;
      else failed++;
    });
    
    console.log(`\nTotal: ${passed} passed, ${failed} failed\n`);
    
    if (failed > 0) {
      process.exit(1);
    }
  }
}

// Usage
const checker = new BudgetChecker('./budget.json');
const report = JSON.parse(readFileSync('./.lighthouseci/lhr.json', 'utf-8'));

checker.check(report).then(() => {
  checker.printResults();
});
```

## Bundle Size Tracking

### Bundlesize Package

```bash
npm install --save-dev bundlesize
```

```json
// package.json
{
  "scripts": {
    "test:bundlesize": "bundlesize"
  },
  "bundlesize": [
    {
      "path": "./build/static/js/main.*.js",
      "maxSize": "100 KB",
      "compression": "gzip"
    },
    {
      "path": "./build/static/js/vendor.*.js",
      "maxSize": "150 KB",
      "compression": "gzip"
    },
    {
      "path": "./build/static/css/main.*.css",
      "maxSize": "30 KB",
      "compression": "gzip"
    },
    {
      "path": "./build/static/css/*.css",
      "maxSize": "50 KB",
      "compression": "gzip"
    }
  ]
}
```

### Size Limit

```bash
npm install --save-dev size-limit @size-limit/preset-app
```

```json
// package.json
{
  "scripts": {
    "size": "size-limit"
  },
  "size-limit": [
    {
      "name": "Main bundle",
      "path": "build/static/js/main.*.js",
      "limit": "100 KB",
      "gzip": true
    },
    {
      "name": "Vendor bundle",
      "path": "build/static/js/vendor.*.js",
      "limit": "150 KB",
      "gzip": true
    },
    {
      "name": "Total JavaScript",
      "path": "build/static/js/*.js",
      "limit": "200 KB",
      "gzip": true
    }
  ]
}
```

### GitHub Actions Integration

```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size

on:
  pull_request:
    branches: [main]

jobs:
  size:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Check bundle size
        run: npm run size
      
      - name: Compare with base
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          build_script: build
```

## webpack-bundle-analyzer

Visualize bundle composition to identify optimization opportunities:

### Setup

```bash
npm install --save-dev webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  // ... other config
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: process.env.ANALYZE ? 'server' : 'disabled',
      openAnalyzer: true,
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json',
      // Set size limits
      defaultSizes: 'gzip'
    })
  ]
};
```

### React (Create React App)

```bash
# Install
npm install --save-dev source-map-explorer

# Add script
# package.json
{
  "scripts": {
    "analyze": "source-map-explorer 'build/static/js/*.js'"
  }
}

# Run
npm run build
npm run analyze
```

### React (Next.js)

```bash
npm install --save-dev @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true'
});

module.exports = withBundleAnalyzer({
  // ... your Next.js config
});

// Run: ANALYZE=true npm run build
```

### Angular

```bash
# Built-in bundle analysis
ng build --stats-json

# Then analyze with webpack-bundle-analyzer
npx webpack-bundle-analyzer dist/your-app/stats.json
```

## Automated Budget Monitoring

### Custom Budget Monitor

```typescript
// budget-monitor.ts
import { promises as fs } from 'fs';
import { gzipSync } from 'zlib';
import { glob } from 'glob';

interface BudgetResult {
  path: string;
  size: number;
  gzipSize: number;
  budget: number;
  exceeded: boolean;
  percentage: number;
}

class BudgetMonitor {
  private budgets: Map<string, number> = new Map();
  
  addBudget(pattern: string, maxSize: number): void {
    this.budgets.set(pattern, maxSize);
  }
  
  async check(): Promise<BudgetResult[]> {
    const results: BudgetResult[] = [];
    
    for (const [pattern, budget] of this.budgets.entries()) {
      const files = await glob(pattern);
      
      for (const file of files) {
        const content = await fs.readFile(file);
        const size = content.length;
        const gzipSize = gzipSync(content).length;
        
        results.push({
          path: file,
          size,
          gzipSize,
          budget,
          exceeded: gzipSize > budget,
          percentage: (gzipSize / budget) * 100
        });
      }
    }
    
    return results;
  }
  
  async report(results: BudgetResult[]): Promise<void> {
    console.log('\n=== Bundle Size Budget Report ===\n');
    
    const table = results.map(r => ({
      File: r.path.split('/').pop() || r.path,
      'Size (gzip)': `${(r.gzipSize / 1024).toFixed(2)} KB`,
      Budget: `${(r.budget / 1024).toFixed(2)} KB`,
      Status: r.exceeded ? '✗ FAIL' : '✓ PASS',
      Usage: `${r.percentage.toFixed(1)}%`
    }));
    
    console.table(table);
    
    const failed = results.filter(r => r.exceeded);
    
    if (failed.length > 0) {
      console.error(`\n${failed.length} file(s) exceeded budget!\n`);
      process.exit(1);
    } else {
      console.log('\n✓ All files within budget\n');
    }
  }
}

// Usage
const monitor = new BudgetMonitor();

monitor.addBudget('build/static/js/main.*.js', 100 * 1024); // 100 KB
monitor.addBudget('build/static/js/vendor.*.js', 150 * 1024); // 150 KB
monitor.addBudget('build/static/css/*.css', 50 * 1024); // 50 KB

monitor.check().then(results => {
  monitor.report(results);
});
```

### Slack/Discord Notifications

```typescript
// notify-budget.ts
interface BudgetNotification {
  status: 'pass' | 'fail';
  results: BudgetResult[];
  totalExceeded: number;
}

async function notifySlack(notification: BudgetNotification): Promise<void> {
  const color = notification.status === 'pass' ? '#36a64f' : '#ff0000';
  
  const message = {
    attachments: [{
      color,
      title: `Performance Budget ${notification.status === 'pass' ? 'Passed' : 'Failed'}`,
      fields: notification.results
        .filter(r => r.exceeded)
        .map(r => ({
          title: r.path,
          value: `${(r.gzipSize / 1024).toFixed(2)} KB / ${(r.budget / 1024).toFixed(2)} KB (${r.percentage.toFixed(1)}%)`,
          short: false
        })),
      footer: `Total exceeded: ${notification.totalExceeded}`,
      ts: Math.floor(Date.now() / 1000)
    }]
  };
  
  await fetch(process.env.SLACK_WEBHOOK_URL!, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(message)
  });
}

// Usage in CI
const results = await monitor.check();
const exceeded = results.filter(r => r.exceeded);

await notifySlack({
  status: exceeded.length === 0 ? 'pass' : 'fail',
  results,
  totalExceeded: exceeded.length
});
```

## Common Mistakes

### 1. Setting Budgets Too Loose

```typescript
// ❌ Bad: Budgets that don't drive improvement
const badBudgets = {
  fcp: 5000,  // Way above "Good" threshold (1.8s)
  lcp: 8000,  // Way above "Good" threshold (2.5s)
  totalJs: 1000 // 1MB is massive for JS
};

// ✅ Good: Aggressive but achievable budgets
const goodBudgets = {
  fcp: 1800,  // Core Web Vitals "Good" threshold
  lcp: 2500,
  totalJs: 200 // KB gzipped
};
```

### 2. Not Enforcing in CI

```bash
# ❌ Bad: Manual checks only
npm run analyze  # Only run locally

# ✅ Good: Automated enforcement
# In CI/CD pipeline:
npm run build
npm run test:bundlesize || exit 1
lhci autorun || exit 1
```

### 3. Ignoring Third-Party Code

```javascript
// ❌ Bad: Only tracking first-party code
budgets: {
  'build/js/main.js': '100 KB'
}

// ✅ Good: Track everything including third-party
budgets: {
  'build/js/main.js': '100 KB',
  'build/js/vendor.js': '150 KB',
  'build/**/*.js': '300 KB' // Total including all third-party
}
```

### 4. No Trend Tracking

```typescript
// ❌ Bad: Only current values
if (currentSize > budget) fail();

// ✅ Good: Track trends over time
const history = await loadBudgetHistory();
history.push({ date: Date.now(), size: currentSize });
await saveBudgetHistory(history);

// Alert on trend
const trend = calculateTrend(history);
if (trend.direction === 'increasing' && trend.rate > 5) {
  alert('Bundle size trending upward!');
}
```

### 5. Same Budgets for All Pages

```javascript
// ❌ Bad: One budget for entire app
budgets: {
  totalJs: 200
}

// ✅ Good: Per-page/route budgets
budgets: {
  'home': { totalJs: 150, lcp: 2000 },
  'dashboard': { totalJs: 300, lcp: 2500 },
  'settings': { totalJs: 100, lcp: 1800 }
}
```

## Best Practices

1. **Set budgets based on user goals** - target Core Web Vitals "Good" thresholds
2. **Enforce in CI/CD** - fail builds that exceed budgets
3. **Track trends over time** - identify regressions early
4. **Different budgets per page/route** - critical pages stricter
5. **Include all resources** - first-party and third-party
6. **Review regularly** - adjust budgets as needed
7. **Visualize with bundle analyzer** - identify optimization opportunities
8. **Alert on violations** - notify team immediately
9. **Document budget rationale** - explain why budgets are set
10. **Start strict, relax if needed** - easier than tightening later

## When to Adjust Budgets

### Tighten Budgets When:
- Successfully meeting current budgets consistently
- User experience metrics improving
- New optimization techniques discovered
- Competing sites performing better

### Relax Budgets When:
- Essential features require more resources
- Target audience has better connectivity
- Business goals prioritize features over speed
- Technical constraints make current budgets impossible

## Interview Questions

### 1. What is a performance budget and why is it important?

**Answer**: A performance budget is a set of constraints that define acceptable performance limits for metrics (FCP, LCP, bundle size, etc.). It's important because: (1) Prevents performance regressions, (2) Forces conscious decisions about new features, (3) Ensures consistent user experience, (4) Aligns team on performance goals, (5) Makes performance measurable and accountable. Without budgets, performance naturally degrades over time as features are added.

### 2. How do you set appropriate performance budgets?

**Answer**: 
1. Analyze current performance (baseline)
2. Research target user environment (devices, connections)
3. Set goals based on Core Web Vitals thresholds (LCP <2.5s, FCP <1.8s)
4. Calculate resource budgets from timing budgets (5s load on 3G = ~600KB gzipped)
5. Benchmark against competitors
6. Start aggressive, adjust based on feasibility
7. Set different budgets per page/route based on importance

### 3. Explain the difference between timing budgets and resource budgets.

**Answer**: Timing budgets set limits on performance metrics (FCP <1.8s, LCP <2.5s, TBT <200ms) - what users experience. Resource budgets limit file sizes and counts (JavaScript <200KB, 50 total requests) - what contributes to those timings. Resource budgets are leading indicators (preventative), timing budgets are lagging indicators (outcome). Both are needed: resource budgets prevent issues, timing budgets measure actual impact.

### 4. How would you enforce performance budgets in CI/CD?

**Answer**: 
1. Lighthouse CI for timing budgets (run on every PR)
2. bundlesize or size-limit for resource budgets
3. Fail build if budgets exceeded
4. Comment on PRs with performance impact
5. Track trends over time
6. Alert team on violations
7. Generate reports for visibility
Use GitHub Actions, GitLab CI, or Jenkins with these tools integrated.

### 5. What is webpack-bundle-analyzer and how does it help with performance budgets?

**Answer**: webpack-bundle-analyzer visualizes bundle composition as an interactive treemap, showing size of every module and dependency. Helps identify: (1) Unexpectedly large dependencies, (2) Duplicate modules, (3) Unused code, (4) Optimization opportunities. Use it to: investigate budget violations, optimize imports, implement code splitting, and make data-driven decisions about dependency changes. Essential for understanding WHY budgets are exceeded, not just THAT they are.

### 6. How do you handle third-party scripts in performance budgets?

**Answer**: 
1. Include in total resource budgets
2. Set separate third-party budget (e.g., <50KB)
3. Use resource-summary:third-party:count metric
4. Load non-critical third-party async/defer
5. Consider self-hosting critical third-party (fonts, analytics)
6. Use facade pattern for heavy embeds (YouTube, maps)
7. Monitor with Request Blocking in DevTools
Challenge: less control over third-party, but they often biggest performance impact.

### 7. What metrics should be included in a comprehensive performance budget?

**Answer**:
Timing: FCP, LCP, TBT/TTI, CLS, FID/INP, TTFB, Speed Index
Resources: Total size (gzipped), JS size, CSS size, Image size, Font size
Counts: Total requests, JS requests, third-party requests
Scores: Lighthouse performance score (>90)
Custom: Time to interactive, First input delay per page
Prioritize Core Web Vitals (LCP, FID/INP, CLS) as they impact rankings.

### 8. How would you track performance budget compliance over time?

**Answer**:
1. Store Lighthouse CI results in permanent storage (LHCI server, database)
2. Log bundle sizes on every build with timestamps
3. Create dashboard visualizing trends (Grafana, custom)
4. Track P50/P75/P95 from RUM data
5. Alert on regressions (>10% degradation)
6. Monthly performance review meetings
7. Correlate with feature releases
8. Track budget "health score" (% of budgets met)
Goal: detect slow degradation before major issues occur.

## Key Takeaways

1. **Performance budgets prevent regressions** - define acceptable limits before problems occur
2. **Set budgets based on Core Web Vitals** - LCP <2.5s, FCP <1.8s, CLS <0.1
3. **Enforce in CI/CD with Lighthouse CI** - fail builds that exceed budgets
4. **Track bundle sizes with automated tools** - bundlesize, size-limit, webpack-bundle-analyzer
5. **Include all resources in budgets** - first-party, third-party, fonts, images
6. **Different budgets per page/route** - stricter for critical landing pages
7. **Monitor trends over time** - detect slow degradation
8. **Alert team on violations** - Slack/email notifications from CI
9. **Start aggressive, adjust as needed** - easier to relax than tighten
10. **Document budget rationale** - explain why limits exist and how they're calculated

## Resources

- [Performance Budgets (web.dev)](https://web.dev/performance-budgets-101/)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [bundlesize](https://github.com/siddharthkp/bundlesize)
- [size-limit](https://github.com/ai/size-limit)
- [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
- [Setting Performance Budgets (Smashing Magazine)](https://www.smashingmagazine.com/2019/07/setting-performance-budgets/)
- [Performance Budget Calculator](https://www.performancebudget.io/)
