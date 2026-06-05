# Router Events in Angular

## The Idea

**In plain English:** Router Events are notifications your app receives every time someone navigates from one page to another — like a play-by-play announcer calling out each step of the journey, from "navigation started" to "navigation finished." You can listen to these announcements and react, for example by showing a loading spinner while the new page is loading.

**Real-world analogy:** Think of a flight departures board at an airport. When a flight is preparing to leave, the board updates through a sequence of statuses — "Boarding," "Gate Closed," "Departed," "Landed." Staff watching the board can react at each stage (a gate agent starts checking tickets when "Boarding" appears; a baggage handler prepares when "Departed" shows up).

- The departures board = the `router.events` stream
- Each status update on the board = a router event (e.g., `NavigationStart`, `NavigationEnd`)
- The staff who watch and react to the board = your component or service that subscribes to router events

---

## Table of Contents

- [Introduction](#introduction)
- [Router Events Overview](#router-events-overview)
- [Navigation Events](#navigation-events)
- [Progress Indicators](#progress-indicators)
- [Event Filtering](#event-filtering)
- [Practical Use Cases](#practical-use-cases)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Router events provide a stream of navigation lifecycle events, allowing you to track, respond to, and control the navigation process. These events are essential for implementing loading indicators, analytics, logging, and navigation guards.

## Router Events Overview

### All Router Event Types

```typescript
// router-events.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, Event,
  NavigationStart,
  NavigationEnd,
  NavigationCancel,
  NavigationError,
  RoutesRecognized,
  GuardsCheckStart,
  GuardsCheckEnd,
  ResolveStart,
  ResolveEnd,
  RouteConfigLoadStart,
  RouteConfigLoadEnd,
  ChildActivationStart,
  ChildActivationEnd,
  ActivationStart,
  ActivationEnd,
  Scroll
} from '@angular/router';

@Component({
  selector: 'app-router-events',
  standalone: true,
  template: `<div>Router Events Demo</div>`
})
export class RouterEventsComponent implements OnInit {
  constructor(private router: Router) {}

  ngOnInit(): void {
    this.router.events.subscribe((event: Event) => {
      if (event instanceof NavigationStart) {
        console.log('NavigationStart:', event);
      }

      if (event instanceof RoutesRecognized) {
        console.log('RoutesRecognized:', event);
      }

      if (event instanceof GuardsCheckStart) {
        console.log('GuardsCheckStart:', event);
      }

      if (event instanceof GuardsCheckEnd) {
        console.log('GuardsCheckEnd:', event);
      }

      if (event instanceof ResolveStart) {
        console.log('ResolveStart:', event);
      }

      if (event instanceof ResolveEnd) {
        console.log('ResolveEnd:', event);
      }

      if (event instanceof RouteConfigLoadStart) {
        console.log('RouteConfigLoadStart:', event);
      }

      if (event instanceof RouteConfigLoadEnd) {
        console.log('RouteConfigLoadEnd:', event);
      }

      if (event instanceof ChildActivationStart) {
        console.log('ChildActivationStart:', event);
      }

      if (event instanceof ChildActivationEnd) {
        console.log('ChildActivationEnd:', event);
      }

      if (event instanceof ActivationStart) {
        console.log('ActivationStart:', event);
      }

      if (event instanceof ActivationEnd) {
        console.log('ActivationEnd:', event);
      }

      if (event instanceof NavigationEnd) {
        console.log('NavigationEnd:', event);
      }

      if (event instanceof NavigationCancel) {
        console.log('NavigationCancel:', event);
      }

      if (event instanceof NavigationError) {
        console.log('NavigationError:', event);
      }

      if (event instanceof Scroll) {
        console.log('Scroll:', event);
      }
    });
  }
}
```

### Event Properties

```typescript
// event-properties.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, NavigationStart, NavigationEnd } from '@angular/router';

@Component({
  selector: 'app-event-properties',
  standalone: true,
  template: `<div>Event Properties</div>`
})
export class EventPropertiesComponent implements OnInit {
  constructor(private router: Router) {}

  ngOnInit(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        console.log('Navigation ID:', event.id);
        console.log('Target URL:', event.url);
        console.log('Navigation Trigger:', event.navigationTrigger);
        console.log('Restore State ID:', event.restoredState?.navigationId);
      }

      if (event instanceof NavigationEnd) {
        console.log('Navigation ID:', event.id);
        console.log('Final URL:', event.url);
        console.log('Final URL after redirects:', event.urlAfterRedirects);
      }
    });
  }
}
```

## Navigation Events

### Basic Navigation Tracking

```typescript
// navigation-tracker.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NavigationTrackerService {
  private navigationInProgress = new BehaviorSubject<boolean>(false);
  navigationInProgress$ = this.navigationInProgress.asObservable();

  private currentUrl = new BehaviorSubject<string>('');
  currentUrl$ = this.currentUrl.asObservable();

  constructor(private router: Router) {
    this.initializeTracking();
  }

  private initializeTracking(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.navigationInProgress.next(true);
      }

      if (event instanceof NavigationEnd) {
        this.navigationInProgress.next(false);
        this.currentUrl.next(event.urlAfterRedirects);
      }

      if (event instanceof NavigationCancel || event instanceof NavigationError) {
        this.navigationInProgress.next(false);
      }
    });
  }
}
```

### Navigation History Service

```typescript
// navigation-history.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

interface NavigationEntry {
  url: string;
  timestamp: Date;
  id: number;
}

@Injectable({ providedIn: 'root' })
export class NavigationHistoryService {
  private history: NavigationEntry[] = [];
  private maxHistoryLength = 10;

  constructor(private router: Router) {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: NavigationEnd) => {
      this.addToHistory({
        url: event.urlAfterRedirects,
        timestamp: new Date(),
        id: event.id
      });
    });
  }

  private addToHistory(entry: NavigationEntry): void {
    this.history.push(entry);
    
    if (this.history.length > this.maxHistoryLength) {
      this.history.shift();
    }
  }

  getHistory(): NavigationEntry[] {
    return [...this.history];
  }

  getPreviousUrl(): string | null {
    if (this.history.length < 2) {
      return null;
    }
    return this.history[this.history.length - 2].url;
  }

  canGoBack(): boolean {
    return this.history.length > 1;
  }

  goBack(): void {
    const previousUrl = this.getPreviousUrl();
    if (previousUrl) {
      this.router.navigateByUrl(previousUrl);
    }
  }
}
```

### Navigation Metrics

```typescript
// navigation-metrics.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError } from '@angular/router';

interface NavigationMetric {
  url: string;
  startTime: number;
  endTime?: number;
  duration?: number;
  success: boolean;
  error?: string;
}

@Injectable({ providedIn: 'root' })
export class NavigationMetricsService {
  private currentMetric: NavigationMetric | null = null;
  private metrics: NavigationMetric[] = [];

  constructor(private router: Router) {
    this.initializeMetrics();
  }

  private initializeMetrics(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.currentMetric = {
          url: event.url,
          startTime: Date.now(),
          success: false
        };
      }

      if (event instanceof NavigationEnd && this.currentMetric) {
        this.currentMetric.endTime = Date.now();
        this.currentMetric.duration = this.currentMetric.endTime - this.currentMetric.startTime;
        this.currentMetric.success = true;
        this.metrics.push({ ...this.currentMetric });
        this.currentMetric = null;
      }

      if (event instanceof NavigationError && this.currentMetric) {
        this.currentMetric.endTime = Date.now();
        this.currentMetric.duration = this.currentMetric.endTime - this.currentMetric.startTime;
        this.currentMetric.success = false;
        this.currentMetric.error = event.error.toString();
        this.metrics.push({ ...this.currentMetric });
        this.currentMetric = null;
      }
    });
  }

  getMetrics(): NavigationMetric[] {
    return [...this.metrics];
  }

  getAverageNavigationTime(): number {
    const successful = this.metrics.filter(m => m.success && m.duration);
    if (successful.length === 0) return 0;
    
    const total = successful.reduce((sum, m) => sum + (m.duration || 0), 0);
    return total / successful.length;
  }

  getSlowestNavigations(count = 5): NavigationMetric[] {
    return [...this.metrics]
      .filter(m => m.success && m.duration)
      .sort((a, b) => (b.duration || 0) - (a.duration || 0))
      .slice(0, count);
  }
}
```

## Progress Indicators

### Loading Spinner Service

```typescript
// loading.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { BehaviorSubject } from 'rxjs';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class LoadingService {
  private loadingSubject = new BehaviorSubject<boolean>(false);
  loading$ = this.loadingSubject.asObservable();

  constructor(private router: Router) {
    this.initializeLoading();
  }

  private initializeLoading(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.loadingSubject.next(true);
      }

      if (
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
      ) {
        this.loadingSubject.next(false);
      }
    });
  }

  show(): void {
    this.loadingSubject.next(true);
  }

  hide(): void {
    this.loadingSubject.next(false);
  }
}

// loading-spinner.component.ts
import { Component } from '@angular/core';
import { LoadingService } from './loading.service';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-loading-spinner',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="loading-overlay" *ngIf="loadingService.loading$ | async">
      <div class="spinner"></div>
      <p>Loading...</p>
    </div>
  `,
  styles: [`
    .loading-overlay {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      z-index: 9999;
    }
    .spinner {
      width: 50px;
      height: 50px;
      border: 5px solid #f3f3f3;
      border-top: 5px solid #3498db;
      border-radius: 50%;
      animation: spin 1s linear infinite;
    }
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
  `]
})
export class LoadingSpinnerComponent {
  constructor(public loadingService: LoadingService) {}
}
```

### Progress Bar Component

```typescript
// progress-bar.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError, Event } from '@angular/router';
import { CommonModule } from '@angular/common';
import { Subject, takeUntil } from 'rxjs';

@Component({
  selector: 'app-progress-bar',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="progress-bar-container" *ngIf="isLoading">
      <div class="progress-bar" [style.width.%]="progress"></div>
    </div>
  `,
  styles: [`
    .progress-bar-container {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 3px;
      background: #f0f0f0;
      z-index: 9999;
    }
    .progress-bar {
      height: 100%;
      background: linear-gradient(to right, #4CAF50, #8BC34A);
      transition: width 0.3s ease;
    }
  `]
})
export class ProgressBarComponent implements OnInit, OnDestroy {
  isLoading = false;
  progress = 0;
  private destroy$ = new Subject<void>();
  private progressInterval: any;

  constructor(private router: Router) {}

  ngOnInit(): void {
    this.router.events
      .pipe(takeUntil(this.destroy$))
      .subscribe((event: Event) => {
        if (event instanceof NavigationStart) {
          this.startProgress();
        }

        if (
          event instanceof NavigationEnd ||
          event instanceof NavigationCancel ||
          event instanceof NavigationError
        ) {
          this.completeProgress();
        }
      });
  }

  private startProgress(): void {
    this.isLoading = true;
    this.progress = 0;

    this.progressInterval = setInterval(() => {
      if (this.progress < 90) {
        this.progress += Math.random() * 10;
      }
    }, 200);
  }

  private completeProgress(): void {
    this.progress = 100;

    setTimeout(() => {
      this.isLoading = false;
      this.progress = 0;
      if (this.progressInterval) {
        clearInterval(this.progressInterval);
      }
    }, 300);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    if (this.progressInterval) {
      clearInterval(this.progressInterval);
    }
  }
}
```

### Delayed Loading Indicator

```typescript
// delayed-loading.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { BehaviorSubject, timer, switchMap, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DelayedLoadingService {
  private loadingSubject = new BehaviorSubject<boolean>(false);
  loading$ = this.loadingSubject.asObservable();
  private loadingTimer: any;
  private readonly DELAY_MS = 300; // Only show loader if navigation takes > 300ms

  constructor(private router: Router) {
    this.initializeLoading();
  }

  private initializeLoading(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        // Delay showing loader
        this.loadingTimer = setTimeout(() => {
          this.loadingSubject.next(true);
        }, this.DELAY_MS);
      }

      if (
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
      ) {
        clearTimeout(this.loadingTimer);
        this.loadingSubject.next(false);
      }
    });
  }
}
```

## Event Filtering

### Filtering Specific Events

```typescript
// filtered-events.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationEnd, Event } from '@angular/router';
import { filter, map } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class FilteredEventsService {
  constructor(private router: Router) {}

  // Only navigation end events
  getNavigationEndEvents(): Observable<NavigationEnd> {
    return this.router.events.pipe(
      filter((event: Event): event is NavigationEnd => event instanceof NavigationEnd)
    );
  }

  // Only successful navigations
  getSuccessfulNavigations(): Observable<string> {
    return this.router.events.pipe(
      filter(event => event instanceof NavigationEnd),
      map((event: NavigationEnd) => event.urlAfterRedirects)
    );
  }

  // Only navigation failures
  getNavigationFailures(): Observable<{ url: string; error: any }> {
    return this.router.events.pipe(
      filter(event => event instanceof NavigationError),
      map((event: NavigationError) => ({
        url: event.url,
        error: event.error
      }))
    );
  }

  // Filter by URL pattern
  getNavigationsToPattern(pattern: RegExp): Observable<NavigationEnd> {
    return this.router.events.pipe(
      filter((event): event is NavigationEnd => event instanceof NavigationEnd),
      filter(event => pattern.test(event.url))
    );
  }
}

// Usage example
@Component({})
export class ExampleComponent implements OnInit {
  constructor(private filteredEvents: FilteredEventsService) {}

  ngOnInit(): void {
    // Track navigations to /products/*
    this.filteredEvents.getNavigationsToPattern(/^\/products\//)
      .subscribe(event => {
        console.log('Navigated to product:', event.url);
      });
  }
}
```

### Custom Event Operators

```typescript
// router-event-operators.ts
import { Router, Event, NavigationStart, NavigationEnd } from '@angular/router';
import { Observable, OperatorFunction } from 'rxjs';
import { filter, map, pairwise, startWith } from 'rxjs/operators';

export function filterNavigationStart(): OperatorFunction<Event, NavigationStart> {
  return filter((event: Event): event is NavigationStart => 
    event instanceof NavigationStart
  );
}

export function filterNavigationEnd(): OperatorFunction<Event, NavigationEnd> {
  return filter((event: Event): event is NavigationEnd => 
    event instanceof NavigationEnd
  );
}

export function mapToUrl(): OperatorFunction<NavigationEnd, string> {
  return map((event: NavigationEnd) => event.urlAfterRedirects);
}

export function detectRouteChange(): OperatorFunction<NavigationEnd, { from: string | null; to: string }> {
  return (source: Observable<NavigationEnd>) => {
    return source.pipe(
      mapToUrl(),
      startWith(null),
      pairwise(),
      map(([from, to]) => ({ from, to: to! }))
    );
  };
}

// Usage
@Injectable({ providedIn: 'root' })
export class RouteChangeService {
  constructor(private router: Router) {
    this.router.events.pipe(
      filterNavigationEnd(),
      detectRouteChange()
    ).subscribe(({ from, to }) => {
      console.log(`Navigated from ${from} to ${to}`);
    });
  }
}
```

## Practical Use Cases

### Analytics Tracking

```typescript
// analytics.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

declare let gtag: Function; // Google Analytics

@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  constructor(private router: Router) {
    this.initializeAnalytics();
  }

  private initializeAnalytics(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: NavigationEnd) => {
      // Google Analytics page view
      if (typeof gtag !== 'undefined') {
        gtag('config', 'GA_MEASUREMENT_ID', {
          page_path: event.urlAfterRedirects
        });
      }

      // Custom analytics
      this.trackPageView(event.urlAfterRedirects);
    });
  }

  private trackPageView(url: string): void {
    console.log('Page view:', url);
    // Send to your analytics service
    // this.http.post('/api/analytics/pageview', { url, timestamp: Date.now() });
  }

  trackEvent(category: string, action: string, label?: string, value?: number): void {
    if (typeof gtag !== 'undefined') {
      gtag('event', action, {
        event_category: category,
        event_label: label,
        value: value
      });
    }
  }
}
```

### Scroll Position Restoration

```typescript
// scroll-position.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, Scroll } from '@angular/router';
import { ViewportScroller } from '@angular/common';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ScrollPositionService {
  private scrollPositions = new Map<string, [number, number]>();

  constructor(
    private router: Router,
    private viewportScroller: ViewportScroller
  ) {
    this.initializeScrollRestoration();
  }

  private initializeScrollRestoration(): void {
    // Save scroll position before navigation
    this.router.events.pipe(
      filter(event => event instanceof NavigationStart)
    ).subscribe(() => {
      const position = this.viewportScroller.getScrollPosition();
      const currentUrl = this.router.url;
      this.scrollPositions.set(currentUrl, position);
    });

    // Restore or reset scroll position after navigation
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: NavigationEnd) => {
      const position = this.scrollPositions.get(event.urlAfterRedirects);
      if (position) {
        setTimeout(() => this.viewportScroller.scrollToPosition(position), 0);
      } else {
        setTimeout(() => this.viewportScroller.scrollToPosition([0, 0]), 0);
      }
    });

    // Handle anchor scrolling
    this.router.events.pipe(
      filter(event => event instanceof Scroll)
    ).subscribe((event: Scroll) => {
      if (event.anchor) {
        setTimeout(() => {
          this.viewportScroller.scrollToAnchor(event.anchor!);
        }, 0);
      }
    });
  }
}
```

### Error Logging Service

```typescript
// error-logging.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationError } from '@angular/router';
import { filter } from 'rxjs/operators';
import { HttpClient } from '@angular/common/http';

interface ErrorLog {
  timestamp: Date;
  url: string;
  error: string;
  stack?: string;
  userAgent: string;
}

@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  constructor(
    private router: Router,
    private http: HttpClient
  ) {
    this.initializeErrorLogging();
  }

  private initializeErrorLogging(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationError)
    ).subscribe((event: NavigationError) => {
      this.logError({
        timestamp: new Date(),
        url: event.url,
        error: event.error.toString(),
        stack: event.error.stack,
        userAgent: navigator.userAgent
      });
    });
  }

  private logError(errorLog: ErrorLog): void {
    console.error('Navigation error:', errorLog);
    
    // Send to backend
    this.http.post('/api/logs/navigation-errors', errorLog)
      .subscribe({
        next: () => console.log('Error logged successfully'),
        error: (err) => console.error('Failed to log error:', err)
      });
  }
}
```

### Route Change Breadcrumbs

```typescript
// breadcrumb.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationEnd, ActivatedRoute } from '@angular/router';
import { filter, map } from 'rxjs/operators';
import { BehaviorSubject } from 'rxjs';

export interface Breadcrumb {
  label: string;
  url: string;
}

@Injectable({ providedIn: 'root' })
export class BreadcrumbService {
  private breadcrumbsSubject = new BehaviorSubject<Breadcrumb[]>([]);
  breadcrumbs$ = this.breadcrumbsSubject.asObservable();

  constructor(
    private router: Router,
    private activatedRoute: ActivatedRoute
  ) {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe(() => {
      const breadcrumbs = this.createBreadcrumbs(this.activatedRoute.root);
      this.breadcrumbsSubject.next(breadcrumbs);
    });
  }

  private createBreadcrumbs(
    route: ActivatedRoute,
    url = '',
    breadcrumbs: Breadcrumb[] = []
  ): Breadcrumb[] {
    const children: ActivatedRoute[] = route.children;

    if (children.length === 0) {
      return breadcrumbs;
    }

    for (const child of children) {
      const routeURL: string = child.snapshot.url
        .map(segment => segment.path)
        .join('/');

      if (routeURL !== '') {
        url += `/${routeURL}`;
      }

      const label = child.snapshot.data['breadcrumb'];
      if (label) {
        breadcrumbs.push({ label, url });
      }

      return this.createBreadcrumbs(child, url, breadcrumbs);
    }

    return breadcrumbs;
  }
}

// breadcrumb.component.ts
@Component({
  selector: 'app-breadcrumb',
  standalone: true,
  imports: [CommonModule, RouterLink],
  template: `
    <nav class="breadcrumb">
      <a routerLink="/">Home</a>
      <span *ngFor="let breadcrumb of breadcrumbs$ | async; let last = last">
        <span class="separator">/</span>
        <a *ngIf="!last" [routerLink]="breadcrumb.url">{{ breadcrumb.label }}</a>
        <span *ngIf="last">{{ breadcrumb.label }}</span>
      </span>
    </nav>
  `
})
export class BreadcrumbComponent {
  breadcrumbs$ = this.breadcrumbService.breadcrumbs$;

  constructor(private breadcrumbService: BreadcrumbService) {}
}
```

## Common Mistakes

### 1. Not Unsubscribing from Router Events

```typescript
// WRONG - Memory leak
@Component({})
export class WrongComponent implements OnInit {
  ngOnInit(): void {
    this.router.events.subscribe(event => {
      // No unsubscribe!
    });
  }
}

// CORRECT - Proper cleanup
@Component({})
export class CorrectComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.router.events
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        // Handle event
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. Not Filtering Events

```typescript
// WRONG - Processes all events
@Component({})
export class WrongComponent {
  constructor(private router: Router) {
    this.router.events.subscribe(event => {
      // This runs for EVERY event type!
      this.doSomething();
    });
  }
}

// CORRECT - Filter specific events
@Component({})
export class CorrectComponent {
  constructor(private router: Router) {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe(event => {
      this.doSomething();
    });
  }
}
```

### 3. Blocking on Every Event

```typescript
// WRONG - Performance issue
@Component({})
export class WrongComponent {
  constructor(private router: Router) {
    this.router.events.subscribe(event => {
      this.performExpensiveOperation(); // On every event!
    });
  }
}

// CORRECT - Only on relevant events
@Component({})
export class CorrectComponent {
  constructor(private router: Router) {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd),
      debounceTime(100)
    ).subscribe(event => {
      this.performExpensiveOperation();
    });
  }
}
```

## Best Practices

### 1. Create Centralized Event Service

```typescript
// router-event.service.ts
@Injectable({ providedIn: 'root' })
export class RouterEventService {
  navigationStart$ = this.router.events.pipe(
    filter(event => event instanceof NavigationStart)
  );

  navigationEnd$ = this.router.events.pipe(
    filter(event => event instanceof NavigationEnd)
  );

  navigationError$ = this.router.events.pipe(
    filter(event => event instanceof NavigationError)
  );

  loading$ = new BehaviorSubject<boolean>(false);

  constructor(private router: Router) {
    this.setupLoadingState();
  }

  private setupLoadingState(): void {
    this.navigationStart$.subscribe(() => this.loading$.next(true));
    
    merge(this.navigationEnd$, this.navigationError$)
      .subscribe(() => this.loading$.next(false));
  }
}
```

### 2. Use Type Guards

```typescript
// Type-safe event filtering
function isNavigationEnd(event: Event): event is NavigationEnd {
  return event instanceof NavigationEnd;
}

this.router.events.pipe(
  filter(isNavigationEnd)
).subscribe(event => {
  // TypeScript knows event is NavigationEnd
  console.log(event.urlAfterRedirects);
});
```

### 3. Debounce Rapid Events

```typescript
// Prevent processing too many events
this.router.events.pipe(
  filter(event => event instanceof NavigationEnd),
  debounceTime(300)
).subscribe(event => {
  // Only process after 300ms of no new events
});
```

## Interview Questions

### Q1: What are router events in Angular?

**Answer:** Router events are a stream of events emitted during the navigation lifecycle, including NavigationStart, NavigationEnd, NavigationCancel, NavigationError, and others. They allow tracking and responding to navigation state changes.

### Q2: How do you show a loading indicator during navigation?

**Answer:** Subscribe to router events, show loader on NavigationStart, and hide on NavigationEnd, NavigationCancel, or NavigationError events.

### Q3: What's the difference between NavigationEnd and RoutesRecognized?

**Answer:** RoutesRecognized fires after URL parsing when routes are identified. NavigationEnd fires after the entire navigation completes, including guards, resolvers, and component activation.

### Q4: How do you track page views for analytics?

**Answer:** Subscribe to NavigationEnd events and send the URL to your analytics service for each successful navigation.

### Q5: Why should you filter router events?

**Answer:** The router emits many events during navigation. Filtering prevents unnecessary processing and improves performance by only handling relevant events.

### Q6: How do you prevent memory leaks with router events?

**Answer:** Use takeUntil pattern with a Subject that completes in ngOnDestroy, or subscribe in a service with providedIn: 'root'.

### Q7: What event indicates lazy module loading?

**Answer:** RouteConfigLoadStart indicates lazy loading begins, and RouteConfigLoadEnd indicates it completes.

### Q8: How do you detect navigation errors?

**Answer:** Subscribe to NavigationError events which provide the URL and error object for failed navigations.

## Key Takeaways

1. **Router events** provide navigation lifecycle visibility
2. **Filter events** to process only relevant ones
3. **NavigationStart/End** are most commonly used
4. **Always unsubscribe** to prevent memory leaks
5. **Use for loading indicators** and progress bars
6. **Track analytics** with NavigationEnd events
7. **Handle errors** with NavigationError events
8. **Centralize event handling** in services
9. **Type guards** improve type safety
10. **Debounce** when processing expensive operations

## Resources

- [Router Events Documentation](https://angular.io/guide/router#router-events)
- [Router Event API](https://angular.io/api/router/Event)
- [NavigationStart API](https://angular.io/api/router/NavigationStart)
- [NavigationEnd API](https://angular.io/api/router/NavigationEnd)
- [Router Events Guide](https://angular.io/guide/router-reference#router-events)
- [RxJS Filter Operator](https://rxjs.dev/api/operators/filter)
