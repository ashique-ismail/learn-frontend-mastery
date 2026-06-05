# Lighthouse Scoring

## The Idea

**In plain English:** Lighthouse is a free tool made by Google that visits your website and gives it a report card with scores from 0 to 100 across categories like speed, accessibility, and search-friendliness — the higher the score, the better your site is doing in that area.

**Real-world analogy:** Think of a health checkup at the doctor's office. The doctor runs several tests (blood pressure, cholesterol, reflexes) and each test gets its own result, then a summary tells you your overall health.

- The doctor's checkup = the Lighthouse audit tool running on your page
- Each individual test (blood pressure, cholesterol) = each performance metric (how fast the page loads, how much it shifts around, how long it blocks you from clicking)
- Your overall health summary score = the final Lighthouse score (0–100)

---

## Overview

Lighthouse is an open-source, automated tool developed by Google for improving the quality of web pages. It audits performance, accessibility, progressive web apps, SEO, and best practices. Understanding how Lighthouse scores your site is crucial for web performance optimization and meeting modern web standards.

## Lighthouse Categories

Lighthouse provides scores across five main categories, each weighted differently and containing multiple audits:

```
Lighthouse Score Breakdown
==========================

┌─────────────────────────────────────────────────────┐
│                 LIGHTHOUSE AUDIT                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Performance (0-100)        Weight: 100%            │
│  ├─ FCP                     10%                     │
│  ├─ LCP                     25%                     │
│  ├─ CLS                     25%                     │
│  ├─ TBT                     30%                     │
│  └─ Speed Index             10%                     │
│                                                      │
│  Accessibility (0-100)      Weight: 100%            │
│  ├─ ARIA                    ~20%                    │
│  ├─ Color Contrast          ~15%                    │
│  ├─ Names and Labels        ~25%                    │
│  └─ Best Practices          ~40%                    │
│                                                      │
│  Best Practices (0-100)     Binary Audits          │
│  ├─ Security                                        │
│  ├─ Modern Standards                                │
│  └─ Browser Errors                                  │
│                                                      │
│  SEO (0-100)                Binary + Weighted       │
│  ├─ Meta Tags                                       │
│  ├─ Mobile Friendly                                 │
│  └─ Crawlability                                    │
│                                                      │
│  PWA (0-100)                Binary Checks           │
│  ├─ Installability                                  │
│  ├─ Service Worker                                  │
│  └─ Offline Support                                 │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## Performance Score Deep Dive

The Performance score is the most complex, using a weighted average of multiple metrics:

```typescript
// Performance Score Calculation
interface PerformanceMetrics {
  FCP: number;  // First Contentful Paint
  LCP: number;  // Largest Contentful Paint
  TBT: number;  // Total Blocking Time
  CLS: number;  // Cumulative Layout Shift
  SI: number;   // Speed Index
}

interface MetricWeight {
  FCP: 0.10;
  LCP: 0.25;
  TBT: 0.30;
  CLS: 0.25;
  SI: 0.10;
}

// Scoring function (simplified)
function calculatePerformanceScore(metrics: PerformanceMetrics): number {
  const weights: MetricWeight = {
    FCP: 0.10,
    LCP: 0.25,
    TBT: 0.30,
    CLS: 0.25,
    SI: 0.10
  };

  // Convert raw metric values to scores (0-100)
  const scores = {
    FCP: metricToScore(metrics.FCP, 1800, 3000),
    LCP: metricToScore(metrics.LCP, 2500, 4000),
    TBT: metricToScore(metrics.TBT, 200, 600),
    CLS: metricToScore(metrics.CLS * 1000, 100, 250), // CLS scaled
    SI: metricToScore(metrics.SI, 3400, 5800)
  };

  // Calculate weighted average
  const performanceScore = 
    scores.FCP * weights.FCP +
    scores.LCP * weights.LCP +
    scores.TBT * weights.TBT +
    scores.CLS * weights.CLS +
    scores.SI * weights.SI;

  return Math.round(performanceScore);
}

// Metric to score conversion using log-normal distribution
function metricToScore(
  value: number,
  median: number,
  falloff: number
): number {
  const distribution = logNormalDistribution(value, median, falloff);
  return Math.round(distribution * 100);
}

function logNormalDistribution(
  value: number,
  median: number,
  falloff: number
): number {
  const location = Math.log(median);
  const shape = Math.log(falloff / median) / Math.sqrt(2);
  
  const standardDeviation = Math.log(value / median) / (Math.sqrt(2) * shape);
  return 1 - (1 + erf(standardDeviation)) / 2;
}

function erf(x: number): number {
  // Error function approximation
  const sign = x >= 0 ? 1 : -1;
  x = Math.abs(x);
  
  const a1 = 0.254829592;
  const a2 = -0.284496736;
  const a3 = 1.421413741;
  const a4 = -1.453152027;
  const a5 = 1.061405429;
  const p = 0.3275911;
  
  const t = 1.0 / (1.0 + p * x);
  const y = 1.0 - (((((a5 * t + a4) * t) + a3) * t + a2) * t + a1) * t * Math.exp(-x * x);
  
  return sign * y;
}

// React: Performance monitoring component
import { useState, useEffect } from 'react';

interface LighthouseScores {
  performance: number | null;
  accessibility: number | null;
  bestPractices: number | null;
  seo: number | null;
  pwa: number | null;
}

function useLighthouseScores(): LighthouseScores {
  const [scores, setScores] = useState<LighthouseScores>({
    performance: null,
    accessibility: null,
    bestPractices: null,
    seo: null,
    pwa: null
  });

  useEffect(() => {
    // In a real app, this would fetch from Lighthouse CI or PageSpeed Insights API
    fetchLighthouseScores().then(setScores);
  }, []);

  return scores;
}

async function fetchLighthouseScores(): Promise<LighthouseScores> {
  try {
    // Example using PageSpeed Insights API
    const url = window.location.href;
    const apiKey = process.env.REACT_APP_PSI_API_KEY;
    const apiUrl = `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=${encodeURIComponent(url)}&key=${apiKey}`;
    
    const response = await fetch(apiUrl);
    const data = await response.json();
    
    const categories = data.lighthouseResult.categories;
    
    return {
      performance: categories.performance.score * 100,
      accessibility: categories.accessibility.score * 100,
      bestPractices: categories['best-practices'].score * 100,
      seo: categories.seo.score * 100,
      pwa: categories.pwa.score * 100
    };
  } catch (error) {
    console.error('Failed to fetch Lighthouse scores:', error);
    return {
      performance: null,
      accessibility: null,
      bestPractices: null,
      seo: null,
      pwa: null
    };
  }
}

// Score display component
function LighthouseScoreCard() {
  const scores = useLighthouseScores();

  const getScoreColor = (score: number | null): string => {
    if (score === null) return 'gray';
    if (score >= 90) return 'green';
    if (score >= 50) return 'orange';
    return 'red';
  };

  const getScoreGrade = (score: number | null): string => {
    if (score === null) return 'N/A';
    if (score >= 90) return 'Good';
    if (score >= 50) return 'Needs Improvement';
    return 'Poor';
  };

  return (
    <div className="lighthouse-scores">
      <h2>Lighthouse Scores</h2>
      <div className="scores-grid">
        {Object.entries(scores).map(([category, score]) => (
          <div key={category} className="score-card">
            <div className="score-category">
              {category.replace(/([A-Z])/g, ' $1').trim()}
            </div>
            <div
              className={`score-value score-${getScoreColor(score)}`}
              style={{ color: getScoreColor(score) }}
            >
              {score === null ? '--' : Math.round(score)}
            </div>
            <div className="score-grade">{getScoreGrade(score)}</div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Angular Implementation:**

```typescript
// lighthouse-scores.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

interface LighthouseScores {
  performance: number | null;
  accessibility: number | null;
  bestPractices: number | null;
  seo: number | null;
  pwa: number | null;
}

@Injectable({
  providedIn: 'root'
})
export class LighthouseScoresService {
  private apiKey = 'YOUR_PSI_API_KEY';

  constructor(private http: HttpClient) {}

  fetchScores(url: string): Observable<LighthouseScores> {
    const apiUrl = `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=${encodeURIComponent(url)}&key=${this.apiKey}`;

    return this.http.get<any>(apiUrl).pipe(
      map(data => {
        const categories = data.lighthouseResult.categories;
        return {
          performance: categories.performance.score * 100,
          accessibility: categories.accessibility.score * 100,
          bestPractices: categories['best-practices'].score * 100,
          seo: categories.seo.score * 100,
          pwa: categories.pwa.score * 100
        };
      }),
      catchError(error => {
        console.error('Failed to fetch Lighthouse scores:', error);
        return of({
          performance: null,
          accessibility: null,
          bestPractices: null,
          seo: null,
          pwa: null
        });
      })
    );
  }

  calculatePerformanceScore(metrics: any): number {
    const weights = {
      FCP: 0.10,
      LCP: 0.25,
      TBT: 0.30,
      CLS: 0.25,
      SI: 0.10
    };

    const scores = {
      FCP: this.metricToScore(metrics.FCP, 1800, 3000),
      LCP: this.metricToScore(metrics.LCP, 2500, 4000),
      TBT: this.metricToScore(metrics.TBT, 200, 600),
      CLS: this.metricToScore(metrics.CLS * 1000, 100, 250),
      SI: this.metricToScore(metrics.SI, 3400, 5800)
    };

    return Math.round(
      scores.FCP * weights.FCP +
      scores.LCP * weights.LCP +
      scores.TBT * weights.TBT +
      scores.CLS * weights.CLS +
      scores.SI * weights.SI
    );
  }

  private metricToScore(value: number, median: number, falloff: number): number {
    // Implementation similar to React version
    return 100; // Simplified
  }
}

// lighthouse-score-card.component.ts
import { Component, OnInit } from '@angular/core';
import { LighthouseScoresService } from './lighthouse-scores.service';

@Component({
  selector: 'app-lighthouse-score-card',
  template: `
    <div class="lighthouse-scores">
      <h2>Lighthouse Scores</h2>
      <div class="scores-grid">
        <div *ngFor="let item of scoresArray" class="score-card">
          <div class="score-category">{{ item.label }}</div>
          <div [class]="'score-value score-' + getScoreColor(item.value)">
            {{ item.value === null ? '--' : item.value }}
          </div>
          <div class="score-grade">{{ getScoreGrade(item.value) }}</div>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .lighthouse-scores {
      padding: 20px;
    }
    .scores-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
      gap: 20px;
    }
    .score-card {
      text-align: center;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    .score-value {
      font-size: 48px;
      font-weight: bold;
      margin: 10px 0;
    }
    .score-green { color: #0cce6b; }
    .score-orange { color: #ffa400; }
    .score-red { color: #ff4e42; }
    .score-gray { color: #999; }
  `]
})
export class LighthouseScoreCardComponent implements OnInit {
  scoresArray: any[] = [];

  constructor(private lighthouseService: LighthouseScoresService) {}

  ngOnInit(): void {
    const currentUrl = window.location.href;
    this.lighthouseService.fetchScores(currentUrl).subscribe(scores => {
      this.scoresArray = [
        { label: 'Performance', value: scores.performance },
        { label: 'Accessibility', value: scores.accessibility },
        { label: 'Best Practices', value: scores.bestPractices },
        { label: 'SEO', value: scores.seo },
        { label: 'PWA', value: scores.pwa }
      ];
    });
  }

  getScoreColor(score: number | null): string {
    if (score === null) return 'gray';
    if (score >= 90) return 'green';
    if (score >= 50) return 'orange';
    return 'red';
  }

  getScoreGrade(score: number | null): string {
    if (score === null) return 'N/A';
    if (score >= 90) return 'Good';
    if (score >= 50) return 'Needs Improvement';
    return 'Poor';
  }
}
```

## Lighthouse CI Integration

Integrate Lighthouse into your CI/CD pipeline to prevent performance regressions:

```yaml
# .github/workflows/lighthouse-ci.yml
name: Lighthouse CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
      
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: lighthouse-results
          path: .lighthouseci
```

**Lighthouse CI Configuration:**

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      // Collect settings
      numberOfRuns: 3,
      startServerCommand: 'npm run serve',
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/about',
        'http://localhost:3000/products'
      ],
      settings: {
        preset: 'desktop',
        throttling: {
          rttMs: 40,
          throughputKbps: 10240,
          cpuSlowdownMultiplier: 1
        }
      }
    },
    assert: {
      // Assertion settings
      preset: 'lighthouse:recommended',
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        
        // Specific metric assertions
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
        
        // Resource assertions
        'unused-javascript': ['warn', { maxLength: 0 }],
        'uses-optimized-images': 'error',
        'modern-image-formats': 'warn',
        'uses-text-compression': 'error',
        'uses-responsive-images': 'warn'
      }
    },
    upload: {
      target: 'temporary-public-storage',
      // Or use LHCI server
      // target: 'lhci',
      // serverBaseUrl: 'https://your-lhci-server.com'
    },
    server: {
      // Optional: LHCI server configuration
    }
  }
};

// React: Component to display CI results
import { useState, useEffect } from 'react';

interface LighthouseCIResult {
  url: string;
  scores: {
    performance: number;
    accessibility: number;
    bestPractices: number;
    seo: number;
  };
  timestamp: string;
}

function LighthouseCIDashboard() {
  const [results, setResults] = useState<LighthouseCIResult[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchCIResults().then(data => {
      setResults(data);
      setLoading(false);
    });
  }, []);

  if (loading) {
    return <div>Loading CI results...</div>;
  }

  return (
    <div className="lighthouse-ci-dashboard">
      <h2>Lighthouse CI Results</h2>
      <div className="results-table">
        <table>
          <thead>
            <tr>
              <th>URL</th>
              <th>Performance</th>
              <th>Accessibility</th>
              <th>Best Practices</th>
              <th>SEO</th>
              <th>Timestamp</th>
            </tr>
          </thead>
          <tbody>
            {results.map((result, index) => (
              <tr key={index}>
                <td>{result.url}</td>
                <td className={getScoreClass(result.scores.performance)}>
                  {result.scores.performance}
                </td>
                <td className={getScoreClass(result.scores.accessibility)}>
                  {result.scores.accessibility}
                </td>
                <td className={getScoreClass(result.scores.bestPractices)}>
                  {result.scores.bestPractices}
                </td>
                <td className={getScoreClass(result.scores.seo)}>
                  {result.scores.seo}
                </td>
                <td>{new Date(result.timestamp).toLocaleString()}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

function getScoreClass(score: number): string {
  if (score >= 90) return 'score-good';
  if (score >= 50) return 'score-moderate';
  return 'score-poor';
}

async function fetchCIResults(): Promise<LighthouseCIResult[]> {
  // Fetch from your LHCI server or storage
  const response = await fetch('/api/lighthouse-ci-results');
  return response.json();
}
```

**Angular Implementation:**

```typescript
// lighthouse-ci-dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface LighthouseCIResult {
  url: string;
  scores: {
    performance: number;
    accessibility: number;
    bestPractices: number;
    seo: number;
  };
  timestamp: string;
}

@Component({
  selector: 'app-lighthouse-ci-dashboard',
  template: `
    <div class="lighthouse-ci-dashboard">
      <h2>Lighthouse CI Results</h2>
      <div class="results-table" *ngIf="!loading">
        <table>
          <thead>
            <tr>
              <th>URL</th>
              <th>Performance</th>
              <th>Accessibility</th>
              <th>Best Practices</th>
              <th>SEO</th>
              <th>Timestamp</th>
            </tr>
          </thead>
          <tbody>
            <tr *ngFor="let result of results">
              <td>{{ result.url }}</td>
              <td [class]="getScoreClass(result.scores.performance)">
                {{ result.scores.performance }}
              </td>
              <td [class]="getScoreClass(result.scores.accessibility)">
                {{ result.scores.accessibility }}
              </td>
              <td [class]="getScoreClass(result.scores.bestPractices)">
                {{ result.scores.bestPractices }}
              </td>
              <td [class]="getScoreClass(result.scores.seo)">
                {{ result.scores.seo }}
              </td>
              <td>{{ result.timestamp | date:'short' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
      <div *ngIf="loading">Loading CI results...</div>
    </div>
  `,
  styles: [`
    table {
      width: 100%;
      border-collapse: collapse;
    }
    th, td {
      padding: 12px;
      text-align: left;
      border-bottom: 1px solid #ddd;
    }
    .score-good { color: #0cce6b; font-weight: bold; }
    .score-moderate { color: #ffa400; font-weight: bold; }
    .score-poor { color: #ff4e42; font-weight: bold; }
  `]
})
export class LighthouseCIDashboardComponent implements OnInit {
  results: LighthouseCIResult[] = [];
  loading = true;

  constructor(private http: HttpClient) {}

  ngOnInit(): void {
    this.http.get<LighthouseCIResult[]>('/api/lighthouse-ci-results')
      .subscribe(data => {
        this.results = data;
        this.loading = false;
      });
  }

  getScoreClass(score: number): string {
    if (score >= 90) return 'score-good';
    if (score >= 50) return 'score-moderate';
    return 'score-poor';
  }
}
```

## Performance Budgets

Set and enforce performance budgets using Lighthouse:

```javascript
// budget.json
[
  {
    "path": "/*",
    "timings": [
      {
        "metric": "first-contentful-paint",
        "budget": 2000
      },
      {
        "metric": "largest-contentful-paint",
        "budget": 2500
      },
      {
        "metric": "cumulative-layout-shift",
        "budget": 0.1
      },
      {
        "metric": "total-blocking-time",
        "budget": 300
      },
      {
        "metric": "speed-index",
        "budget": 3400
      }
    ],
    "resourceSizes": [
      {
        "resourceType": "script",
        "budget": 300
      },
      {
        "resourceType": "stylesheet",
        "budget": 50
      },
      {
        "resourceType": "image",
        "budget": 500
      },
      {
        "resourceType": "font",
        "budget": 100
      },
      {
        "resourceType": "document",
        "budget": 50
      },
      {
        "resourceType": "total",
        "budget": 1000
      }
    ],
    "resourceCounts": [
      {
        "resourceType": "script",
        "budget": 15
      },
      {
        "resourceType": "stylesheet",
        "budget": 5
      },
      {
        "resourceType": "image",
        "budget": 30
      },
      {
        "resourceType": "third-party",
        "budget": 10
      }
    ]
  }
]

// webpack.config.js - Enforce budgets during build
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  // ... other config
  performance: {
    maxEntrypointSize: 300000, // 300KB
    maxAssetSize: 200000, // 200KB
    hints: 'error'
  },
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html',
      defaultSizes: 'gzip',
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json'
    })
  ]
};

// React: Budget monitoring component
import { useState, useEffect } from 'react';

interface BudgetStatus {
  metric: string;
  current: number;
  budget: number;
  status: 'pass' | 'warn' | 'fail';
  percentage: number;
}

function PerformanceBudgetDashboard() {
  const [budgets, setBudgets] = useState<BudgetStatus[]>([]);

  useEffect(() => {
    checkBudgets().then(setBudgets);
  }, []);

  return (
    <div className="budget-dashboard">
      <h2>Performance Budgets</h2>
      <div className="budget-list">
        {budgets.map(budget => (
          <div key={budget.metric} className={`budget-item budget-${budget.status}`}>
            <div className="budget-metric">{budget.metric}</div>
            <div className="budget-values">
              <span className="budget-current">{budget.current}</span>
              <span className="budget-separator">/</span>
              <span className="budget-target">{budget.budget}</span>
            </div>
            <div className="budget-bar">
              <div
                className="budget-bar-fill"
                style={{ width: `${Math.min(budget.percentage, 100)}%` }}
              />
            </div>
            <div className="budget-percentage">{budget.percentage.toFixed(1)}%</div>
          </div>
        ))}
      </div>
    </div>
  );
}

async function checkBudgets(): Promise<BudgetStatus[]> {
  // Fetch current metrics
  const metrics = await getCurrentMetrics();
  
  // Load budgets from configuration
  const budgetConfig = [
    { metric: 'FCP', budget: 2000 },
    { metric: 'LCP', budget: 2500 },
    { metric: 'CLS', budget: 0.1 },
    { metric: 'TBT', budget: 300 },
    { metric: 'Bundle Size', budget: 300 }
  ];

  return budgetConfig.map(({ metric, budget }) => {
    const current = metrics[metric] || 0;
    const percentage = (current / budget) * 100;
    
    let status: 'pass' | 'warn' | 'fail';
    if (percentage <= 80) status = 'pass';
    else if (percentage <= 100) status = 'warn';
    else status = 'fail';

    return { metric, current, budget, status, percentage };
  });
}

async function getCurrentMetrics(): Promise<Record<string, number>> {
  // Fetch from monitoring service
  return {
    'FCP': 1500,
    'LCP': 2200,
    'CLS': 0.08,
    'TBT': 250,
    'Bundle Size': 280
  };
}
```

## Lighthouse Audit Flow

```
Lighthouse Audit Process
=========================

┌─────────────────────────────────────────┐
│  1. Navigation & Page Load              │
│     - Navigate to URL                   │
│     - Wait for page load                │
│     - Collect network requests          │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  2. Gather Phase                        │
│     - Collect traces                    │
│     - Capture screenshots               │
│     - Record network activity           │
│     - Measure timing metrics            │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  3. Audit Phase                         │
│     - Run 50+ audits                    │
│     - Check accessibility               │
│     - Analyze performance               │
│     - Validate best practices           │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  4. Scoring Phase                       │
│     - Calculate category scores         │
│     - Apply weights to metrics          │
│     - Generate recommendations          │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  5. Report Generation                   │
│     - Create JSON report                │
│     - Generate HTML report              │
│     - Provide optimization tips         │
└─────────────────────────────────────────┘
```

## Programmatic Lighthouse Usage

```typescript
// lighthouse-runner.ts
import lighthouse from 'lighthouse';
import * as chromeLauncher from 'chrome-launcher';
import { writeFileSync } from 'fs';

interface LighthouseOptions {
  port?: number;
  output?: 'json' | 'html' | 'csv';
  onlyCategories?: string[];
  throttlingMethod?: 'simulate' | 'devtools';
}

async function runLighthouse(
  url: string,
  options: LighthouseOptions = {}
): Promise<any> {
  // Launch Chrome
  const chrome = await chromeLauncher.launch({
    chromeFlags: ['--headless', '--disable-gpu', '--no-sandbox']
  });

  const lighthouseOptions = {
    port: chrome.port,
    output: options.output || 'json',
    onlyCategories: options.onlyCategories || [
      'performance',
      'accessibility',
      'best-practices',
      'seo'
    ],
    throttlingMethod: options.throttlingMethod || 'simulate'
  };

  // Run Lighthouse
  const runnerResult = await lighthouse(url, lighthouseOptions);

  // Kill Chrome
  await chrome.kill();

  return runnerResult;
}

// Usage example
async function auditWebsite() {
  const url = 'https://example.com';
  
  console.log('Running Lighthouse audit...');
  const result = await runLighthouse(url, {
    output: 'json',
    onlyCategories: ['performance']
  });

  const scores = {
    performance: result.lhr.categories.performance.score * 100,
    fcp: result.lhr.audits['first-contentful-paint'].numericValue,
    lcp: result.lhr.audits['largest-contentful-paint'].numericValue,
    cls: result.lhr.audits['cumulative-layout-shift'].numericValue,
    tbt: result.lhr.audits['total-blocking-time'].numericValue
  };

  console.log('Lighthouse Results:', scores);

  // Save report
  writeFileSync(
    'lighthouse-report.json',
    JSON.stringify(result.lhr, null, 2)
  );

  return scores;
}

// Batch auditing multiple URLs
async function batchAudit(urls: string[]): Promise<void> {
  const results = [];

  for (const url of urls) {
    console.log(`Auditing: ${url}`);
    const result = await runLighthouse(url);
    
    results.push({
      url,
      timestamp: new Date().toISOString(),
      scores: {
        performance: result.lhr.categories.performance.score * 100,
        accessibility: result.lhr.categories.accessibility.score * 100,
        bestPractices: result.lhr.categories['best-practices'].score * 100,
        seo: result.lhr.categories.seo.score * 100
      }
    });

    // Wait between audits
    await new Promise(resolve => setTimeout(resolve, 5000));
  }

  // Save batch results
  writeFileSync(
    'batch-audit-results.json',
    JSON.stringify(results, null, 2)
  );

  console.log('Batch audit complete!');
}

// Run audits
auditWebsite().catch(console.error);
```

## Common Mistakes

1. **Only testing on fast networks/devices**
   - Problem: Production users have slower connections
   - Solution: Test with throttling enabled

2. **Ignoring mobile scores**
   - Problem: Mobile often scores worse than desktop
   - Solution: Prioritize mobile-first optimization

3. **Not running multiple audits**
   - Problem: Single run can have variance
   - Solution: Run 3-5 audits and average results

4. **Testing only the homepage**
   - Problem: Other pages may have different performance
   - Solution: Audit key user journeys

5. **Not setting performance budgets**
   - Problem: Performance regression goes unnoticed
   - Solution: Enforce budgets in CI/CD

6. **Optimizing only for Lighthouse**
   - Problem: Real user experience may differ
   - Solution: Use RUM alongside Lighthouse

7. **Ignoring third-party scripts**
   - Problem: Third parties often cause poor scores
   - Solution: Audit and optimize third-party loading

8. **Not acting on recommendations**
   - Problem: Auditing without improvement
   - Solution: Create actionable improvement plan

## Best Practices

1. **Automate audits**: Integrate into CI/CD pipeline
2. **Set thresholds**: Define minimum acceptable scores
3. **Track trends**: Monitor scores over time
4. **Test multiple pages**: Audit representative pages
5. **Use realistic conditions**: Enable throttling
6. **Run multiple times**: Average results for accuracy
7. **Prioritize fixes**: Focus on highest-impact improvements
8. **Document baselines**: Track improvements from baseline
9. **Share results**: Make scores visible to team
10. **Continuous monitoring**: Regular audits, not one-time checks

## When to Use

**Run Lighthouse when:**
- Deploying new features
- Making performance optimizations
- Conducting accessibility audits
- SEO optimization work
- Pre-production testing
- Regular health checks

**May not be needed for:**
- Internal admin tools
- Development environments
- API-only services
- Non-web applications

## Interview Questions

1. **How does Lighthouse calculate the Performance score?**
   - Weighted average of 5 metrics: FCP (10%), LCP (25%), TBT (30%), CLS (25%), SI (10%)
   - Each metric converted to 0-100 score using log-normal distribution
   - Final score is weighted sum

2. **What's the difference between lab data and field data?**
   - Lab data: Controlled environment (Lighthouse), consistent, reproducible
   - Field data: Real users (CrUX), variable conditions, actual experience
   - Both provide complementary insights

3. **How would you integrate Lighthouse into a CI/CD pipeline?**
   - Use Lighthouse CI tool
   - Configure lighthouserc.js with assertions
   - Add GitHub Actions/GitLab CI workflow
   - Set score thresholds to fail builds
   - Store historical results

4. **What are performance budgets and why are they important?**
   - Limits on metrics/resource sizes
   - Prevent performance regressions
   - Keep teams accountable
   - Help prioritize optimization work
   - Can be enforced automatically

5. **How do you debug a poor Lighthouse score?**
   - Review individual audit failures
   - Check waterfall for bottlenecks
   - Analyze bundle size
   - Examine third-party scripts
   - Test with throttling enabled

6. **What's the relationship between Lighthouse and Core Web Vitals?**
   - Lighthouse measures Core Web Vitals metrics
   - Uses TBT as proxy for FID/INP
   - Performance score heavily weights Core Web Vitals
   - Both focus on user experience

7. **How do different Lighthouse modes (mobile vs desktop) affect scores?**
   - Mobile: 4x CPU slowdown, throttled network
   - Desktop: Faster settings, higher thresholds
   - Mobile typically scores lower
   - Both important for different contexts

8. **What's the purpose of the Lighthouse scoring algorithm's log-normal distribution?**
   - Reflects real-world metric distributions
   - Makes scores more sensitive to improvements
   - Prevents linear scoring issues
   - Based on HTTP Archive data

## Key Takeaways

1. Lighthouse scores across 5 categories: Performance, Accessibility, Best Practices, SEO, PWA
2. Performance score is weighted: TBT (30%), LCP (25%), CLS (25%), FCP (10%), SI (10%)
3. Score ranges: 90-100 (good), 50-89 (needs improvement), 0-49 (poor)
4. Integrate Lighthouse CI into your deployment pipeline
5. Set and enforce performance budgets to prevent regressions
6. Run multiple audits and average results for accuracy
7. Test with throttling enabled to simulate real conditions
8. Prioritize mobile optimization as it typically scores worse
9. Use both lab data (Lighthouse) and field data (RUM) for complete picture
10. Lighthouse is a diagnostic tool, not the end goal - real user experience matters most

## Resources

- **Official Documentation**
  - [Lighthouse Documentation](https://developer.chrome.com/docs/lighthouse/)
  - [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
  - [Lighthouse Scoring Guide](https://web.dev/performance-scoring/)

- **Tools**
  - [PageSpeed Insights](https://pagespeed.web.dev/)
  - [Lighthouse Chrome Extension](https://chrome.google.com/webstore/detail/lighthouse)
  - [Lighthouse CLI](https://www.npmjs.com/package/lighthouse)

- **Articles**
  - [Lighthouse Performance Scoring](https://web.dev/performance-scoring/)
  - [Setting Performance Budgets](https://web.dev/performance-budgets-101/)
  - [Lighthouse User Flows](https://web.dev/lighthouse-user-flows/)
