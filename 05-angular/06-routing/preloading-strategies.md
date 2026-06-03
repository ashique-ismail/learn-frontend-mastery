# Preloading Strategies in Angular

## Table of Contents
- [Introduction](#introduction)
- [PreloadAllModules Strategy](#preloadallmodules-strategy)
- [NoPreloading Strategy](#nopreloading-strategy)
- [Custom Preloading Strategies](#custom-preloading-strategies)
- [Network-Aware Preloading](#network-aware-preloading)
- [Selective Preloading](#selective-preloading)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Preloading strategies determine when and how lazy-loaded modules are fetched. Angular provides built-in strategies and allows custom implementations to optimize application loading based on network conditions, user behavior, and business logic.

## PreloadAllModules Strategy

### Basic Configuration

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules)
    )
  ]
};

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES)
  },
  {
    path: 'products',
    loadChildren: () => import('./products/products.routes')
      .then(m => m.PRODUCT_ROUTES)
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.routes')
      .then(m => m.REPORT_ROUTES)
  }
];

// All lazy modules will be preloaded after initial navigation
```

### When to Use PreloadAllModules

```typescript
/**
 * Use PreloadAllModules when:
 * 1. Your application has few lazy-loaded modules
 * 2. Modules are relatively small
 * 3. Users typically access most routes in a session
 * 4. Network bandwidth is generally good
 * 5. You want the simplest preloading setup
 * 
 * Avoid when:
 * 1. You have many large lazy modules
 * 2. Some routes are rarely accessed
 * 3. Mobile/slow network users are common
 * 4. Initial load time is critical
 */

// Example: Small application with 3-4 feature modules
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules)
    )
  ]
};
```

## NoPreloading Strategy

### Basic Configuration

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withPreloading, NoPreloading } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(NoPreloading) // Explicit no preloading
    )
  ]
};

// Or simply don't provide withPreloading
export const appConfigNoPreload: ApplicationConfig = {
  providers: [
    provideRouter(routes) // No preloading by default
  ]
};

/**
 * Use NoPreloading when:
 * 1. You want maximum control over loading
 * 2. You'll implement custom preloading logic
 * 3. Bandwidth is limited
 * 4. You want modules loaded strictly on-demand
 */
```

## Custom Preloading Strategies

### Basic Custom Strategy

```typescript
// selective-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Check if route has preload flag
    if (route.data && route.data['preload']) {
      console.log('Preloading:', route.path);
      return load();
    }
    
    return of(null);
  }
}

// app.config.ts
import { provideRouter, withPreloading } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(SelectivePreloadStrategy)
    )
  ]
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes'),
    data: { preload: true } // Will be preloaded
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
    // No preload flag - won't be preloaded
  },
  {
    path: 'settings',
    loadChildren: () => import('./settings/settings.routes'),
    data: { preload: true } // Will be preloaded
  }
];
```

### Delayed Preloading Strategy

```typescript
// delayed-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class DelayedPreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const delay = route.data?.['preloadDelay'] || 0;
    
    if (delay > 0) {
      console.log(`Preloading ${route.path} after ${delay}ms`);
      return timer(delay).pipe(
        switchMap(() => load())
      );
    }
    
    if (route.data?.['preload']) {
      return load();
    }
    
    return of(null);
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes'),
    data: { preload: true } // Immediate preload
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.routes'),
    data: { preload: true, preloadDelay: 2000 } // Preload after 2 seconds
  },
  {
    path: 'analytics',
    loadChildren: () => import('./analytics/analytics.routes'),
    data: { preload: true, preloadDelay: 5000 } // Preload after 5 seconds
  }
];
```

### Priority-Based Preloading

```typescript
// priority-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, concat, EMPTY } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

interface RouteWithPriority {
  route: Route;
  load: () => Observable<any>;
  priority: number;
}

@Injectable({ providedIn: 'root' })
export class PriorityPreloadStrategy implements PreloadingStrategy {
  private routesToPreload: RouteWithPriority[] = [];

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const priority = route.data?.['preloadPriority'];
    
    if (priority !== undefined) {
      this.routesToPreload.push({ route, load, priority });
      
      // Sort by priority (higher first)
      this.routesToPreload.sort((a, b) => b.priority - a.priority);
      
      return EMPTY;
    }
    
    return of(null);
  }

  startPreloading(): void {
    // Preload routes in priority order
    const preloadObservables = this.routesToPreload.map(item => {
      console.log(`Preloading ${item.route.path} (priority: ${item.priority})`);
      return item.load();
    });

    concat(...preloadObservables).subscribe();
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes'),
    data: { preloadPriority: 10 } // Highest priority
  },
  {
    path: 'profile',
    loadChildren: () => import('./profile/profile.routes'),
    data: { preloadPriority: 8 }
  },
  {
    path: 'settings',
    loadChildren: () => import('./settings/settings.routes'),
    data: { preloadPriority: 5 }
  },
  {
    path: 'help',
    loadChildren: () => import('./help/help.routes'),
    data: { preloadPriority: 1 } // Lowest priority
  }
];

// app.component.ts
@Component({})
export class AppComponent implements OnInit {
  constructor(private preloadStrategy: PriorityPreloadStrategy) {}

  ngOnInit(): void {
    // Start preloading after app initializes
    setTimeout(() => {
      this.preloadStrategy.startPreloading();
    }, 1000);
  }
}
```

## Network-Aware Preloading

### Connection Speed Detection

```typescript
// network-aware-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NetworkAwarePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (!route.data?.['preload']) {
      return of(null);
    }

    const connection = this.getConnectionInfo();
    
    // Don't preload on slow connections
    if (connection.saveData || connection.effectiveType === 'slow-2g' || connection.effectiveType === '2g') {
      console.log(`Skipping preload of ${route.path} due to slow connection`);
      return of(null);
    }

    // Preload on fast connections
    if (connection.effectiveType === '4g' || connection.downlink > 1.5) {
      console.log(`Preloading ${route.path} on fast connection`);
      return load();
    }

    // Conditionally preload on 3g
    if (connection.effectiveType === '3g') {
      const requiredBandwidth = route.data['minBandwidth'] || 0.5;
      if (connection.downlink >= requiredBandwidth) {
        console.log(`Preloading ${route.path} on acceptable 3g connection`);
        return load();
      }
    }

    return of(null);
  }

  private getConnectionInfo(): any {
    const nav: any = navigator;
    const connection = nav.connection || nav.mozConnection || nav.webkitConnection;
    
    return {
      effectiveType: connection?.effectiveType || '4g',
      downlink: connection?.downlink || 10,
      saveData: connection?.saveData || false
    };
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes'),
    data: { preload: true } // Preload on good connections
  },
  {
    path: 'large-reports',
    loadChildren: () => import('./reports/reports.routes'),
    data: { preload: true, minBandwidth: 2.0 } // Only on fast connections
  }
];
```

### Data Saver Mode Detection

```typescript
// data-saver-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DataSaverPreloadStrategy implements PreloadingStrategy {
  private isDataSaverEnabled(): boolean {
    const nav: any = navigator;
    const connection = nav.connection || nav.mozConnection || nav.webkitConnection;
    return connection?.saveData || false;
  }

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Never preload in data saver mode
    if (this.isDataSaverEnabled()) {
      console.log('Data saver enabled - skipping all preloading');
      return of(null);
    }

    // Preload if explicitly marked
    if (route.data?.['preload']) {
      console.log(`Preloading ${route.path}`);
      return load();
    }

    return of(null);
  }
}
```

### Bandwidth-Based Strategy

```typescript
// bandwidth-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class BandwidthPreloadStrategy implements PreloadingStrategy {
  private readonly BANDWIDTH_THRESHOLDS = {
    LOW: 0.5,    // < 0.5 Mbps - no preloading
    MEDIUM: 1.5, // 0.5-1.5 Mbps - selective preloading
    HIGH: 3.0    // > 3.0 Mbps - preload all
  };

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (!route.data?.['preload']) {
      return of(null);
    }

    const bandwidth = this.getCurrentBandwidth();
    const routePriority = route.data['priority'] || 'medium';

    console.log(`Current bandwidth: ${bandwidth} Mbps, Route priority: ${routePriority}`);

    // High bandwidth - preload everything
    if (bandwidth >= this.BANDWIDTH_THRESHOLDS.HIGH) {
      console.log(`Preloading ${route.path} (high bandwidth)`);
      return load();
    }

    // Medium bandwidth - only high priority routes
    if (bandwidth >= this.BANDWIDTH_THRESHOLDS.MEDIUM && routePriority === 'high') {
      console.log(`Preloading ${route.path} (medium bandwidth, high priority)`);
      return load();
    }

    // Low bandwidth - no preloading
    console.log(`Skipping ${route.path} (insufficient bandwidth)`);
    return of(null);
  }

  private getCurrentBandwidth(): number {
    const nav: any = navigator;
    const connection = nav.connection || nav.mozConnection || nav.webkitConnection;
    return connection?.downlink || 10; // Default to 10 Mbps if unknown
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes'),
    data: { preload: true, priority: 'high' }
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.routes'),
    data: { preload: true, priority: 'medium' }
  },
  {
    path: 'archive',
    loadChildren: () => import('./archive/archive.routes'),
    data: { preload: true, priority: 'low' }
  }
];
```

## Selective Preloading

### User Behavior-Based Preloading

```typescript
// behavior-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class BehaviorPreloadStrategy implements PreloadingStrategy {
  private userActivity: Date[] = [];
  private readonly INACTIVITY_THRESHOLD = 2000; // 2 seconds of inactivity

  constructor() {
    this.trackUserActivity();
  }

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (!route.data?.['preload']) {
      return of(null);
    }

    // Wait for user inactivity before preloading
    return timer(this.INACTIVITY_THRESHOLD).pipe(
      switchMap(() => {
        if (this.isUserInactive()) {
          console.log(`Preloading ${route.path} during inactivity`);
          return load();
        }
        return of(null);
      })
    );
  }

  private trackUserActivity(): void {
    ['click', 'scroll', 'keypress', 'mousemove'].forEach(eventType => {
      document.addEventListener(eventType, () => {
        this.userActivity.push(new Date());
        // Keep only last 10 activities
        if (this.userActivity.length > 10) {
          this.userActivity.shift();
        }
      });
    });
  }

  private isUserInactive(): boolean {
    if (this.userActivity.length === 0) {
      return true;
    }

    const lastActivity = this.userActivity[this.userActivity.length - 1];
    const timeSinceLastActivity = Date.now() - lastActivity.getTime();
    
    return timeSinceLastActivity > this.INACTIVITY_THRESHOLD;
  }
}
```

### Hover-Based Preloading

```typescript
// hover-preload.service.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class HoverPreloadService {
  private preloadedRoutes = new Set<string>();

  constructor(private router: Router) {}

  preloadOnHover(path: string): void {
    if (this.preloadedRoutes.has(path)) {
      console.log(`${path} already preloaded`);
      return;
    }

    const route = this.router.config.find(r => r.path === path);
    
    if (route && route.loadChildren) {
      console.log(`Preloading ${path} on hover`);
      
      // Call loadChildren to trigger module load
      if (typeof route.loadChildren === 'function') {
        route.loadChildren().then(() => {
          this.preloadedRoutes.add(path);
          console.log(`${path} preloaded successfully`);
        }).catch(err => {
          console.error(`Failed to preload ${path}:`, err);
        });
      }
    }
  }
}

// Usage in component
@Component({
  selector: 'app-nav',
  template: `
    <nav>
      <a 
        routerLink="/dashboard"
        (mouseenter)="preloadRoute('dashboard')">
        Dashboard
      </a>
      <a 
        routerLink="/reports"
        (mouseenter)="preloadRoute('reports')">
        Reports
      </a>
    </nav>
  `
})
export class NavComponent {
  constructor(private hoverPreload: HoverPreloadService) {}

  preloadRoute(path: string): void {
    this.hoverPreload.preloadOnHover(path);
  }
}
```

### Predictive Preloading

```typescript
// predictive-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route, Router, NavigationEnd } from '@angular/router';
import { Observable, of } from 'rxjs';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class PredictivePreloadStrategy implements PreloadingStrategy {
  private routeHistory: string[] = [];
  private routePatterns = new Map<string, string[]>(); // Common paths after each route

  constructor(private router: Router) {
    this.trackNavigationPatterns();
  }

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Always return of(null) initially
    // Actual preloading happens based on predictions
    return of(null);
  }

  private trackNavigationPatterns(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: NavigationEnd) => {
      const currentPath = event.urlAfterRedirects;
      
      if (this.routeHistory.length > 0) {
        const previousPath = this.routeHistory[this.routeHistory.length - 1];
        
        // Record pattern: previous -> current
        if (!this.routePatterns.has(previousPath)) {
          this.routePatterns.set(previousPath, []);
        }
        this.routePatterns.get(previousPath)!.push(currentPath);
      }

      this.routeHistory.push(currentPath);
      
      // Predict and preload likely next routes
      this.preloadPredictedRoutes(currentPath);
    });
  }

  private preloadPredictedRoutes(currentPath: string): void {
    const predictions = this.routePatterns.get(currentPath);
    
    if (!predictions || predictions.length === 0) {
      return;
    }

    // Find most common next route
    const frequency = new Map<string, number>();
    predictions.forEach(path => {
      frequency.set(path, (frequency.get(path) || 0) + 1);
    });

    const mostLikely = Array.from(frequency.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, 2) // Preload top 2 predictions
      .map(([path]) => path);

    mostLikely.forEach(path => {
      console.log(`Predictively preloading: ${path}`);
      this.preloadPath(path);
    });
  }

  private preloadPath(path: string): void {
    // Find and load the route
    const route = this.router.config.find(r => `/${r.path}` === path);
    
    if (route && route.loadChildren && typeof route.loadChildren === 'function') {
      route.loadChildren().catch(err => {
        console.error(`Failed to preload ${path}:`, err);
      });
    }
  }
}
```

## Advanced Patterns

### Quota-Based Preloading

```typescript
// quota-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class QuotaPreloadStrategy implements PreloadingStrategy {
  private totalPreloaded = 0;
  private readonly MAX_PRELOAD_MB = 10; // Maximum 10MB to preload

  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (!route.data?.['preload']) {
      return of(null);
    }

    const estimatedSize = route.data['estimatedSize'] || 1; // MB
    
    if (this.totalPreloaded + estimatedSize > this.MAX_PRELOAD_MB) {
      console.log(`Quota exceeded, skipping ${route.path}`);
      return of(null);
    }

    console.log(`Preloading ${route.path} (${estimatedSize}MB)`);
    this.totalPreloaded += estimatedSize;
    
    return load();
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes'),
    data: { preload: true, estimatedSize: 2 } // 2MB
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.routes'),
    data: { preload: true, estimatedSize: 5 } // 5MB
  },
  {
    path: 'analytics',
    loadChildren: () => import('./analytics/analytics.routes'),
    data: { preload: true, estimatedSize: 4 } // Would exceed quota
  }
];
```

### Time-Window Preloading

```typescript
// time-window-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class TimeWindowPreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (!route.data?.['preload']) {
      return of(null);
    }

    const startHour = route.data['preloadStartHour'];
    const endHour = route.data['preloadEndHour'];

    if (startHour !== undefined && endHour !== undefined) {
      if (this.isInTimeWindow(startHour, endHour)) {
        console.log(`Preloading ${route.path} (within time window)`);
        return load();
      } else {
        console.log(`Skipping ${route.path} (outside time window)`);
        return of(null);
      }
    }

    return load();
  }

  private isInTimeWindow(startHour: number, endHour: number): boolean {
    const currentHour = new Date().getHours();
    return currentHour >= startHour && currentHour < endHour;
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'business-hours',
    loadChildren: () => import('./business/business.routes'),
    data: { 
      preload: true, 
      preloadStartHour: 9, 
      preloadEndHour: 17 
    }
  },
  {
    path: 'night-mode',
    loadChildren: () => import('./night/night.routes'),
    data: { 
      preload: true, 
      preloadStartHour: 20, 
      preloadEndHour: 6 
    }
  }
];
```

## Common Mistakes

### 1. Preloading Everything on Slow Connections

```typescript
// WRONG - No network consideration
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules))
  ]
};

// CORRECT - Network-aware preloading
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withPreloading(NetworkAwarePreloadStrategy))
  ]
};
```

### 2. Not Considering Module Size

```typescript
// WRONG - No size consideration
@Injectable({ providedIn: 'root' })
export class WrongStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}

// CORRECT - Size-aware preloading
@Injectable({ providedIn: 'root' })
export class CorrectStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const size = route.data?.['estimatedSize'] || 0;
    const preload = route.data?.['preload'];
    
    if (preload && size < 5) { // Only preload if < 5MB
      return load();
    }
    return of(null);
  }
}
```

### 3. Forgetting to Return Observable

```typescript
// WRONG - Not returning Observable
@Injectable({ providedIn: 'root' })
export class WrongStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.['preload']) {
      load(); // Forgot to return!
    }
    return of(null);
  }
}

// CORRECT - Always return Observable
@Injectable({ providedIn: 'root' })
export class CorrectStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

## Best Practices

### 1. Combine Multiple Strategies

```typescript
// combined-preload.strategy.ts
@Injectable({ providedIn: 'root' })
export class CombinedPreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // 1. Check if preload is enabled
    if (!route.data?.['preload']) {
      return of(null);
    }

    // 2. Check network conditions
    if (this.isSlowConnection()) {
      return of(null);
    }

    // 3. Check priority
    const priority = route.data['priority'] || 'low';
    if (priority === 'low' && !this.isUserIdle()) {
      return of(null);
    }

    // 4. Check quota
    const size = route.data['estimatedSize'] || 1;
    if (!this.hasQuotaFor(size)) {
      return of(null);
    }

    // All checks passed - preload
    return load();
  }

  private isSlowConnection(): boolean {
    // Implementation
    return false;
  }

  private isUserIdle(): boolean {
    // Implementation
    return true;
  }

  private hasQuotaFor(size: number): boolean {
    // Implementation
    return true;
  }
}
```

### 2. Monitor Preloading Performance

```typescript
// preload-monitor.service.ts
@Injectable({ providedIn: 'root' })
export class PreloadMonitorService {
  private preloadMetrics = new Map<string, {
    startTime: number;
    endTime?: number;
    success: boolean;
  }>();

  logPreloadStart(path: string): void {
    this.preloadMetrics.set(path, {
      startTime: Date.now(),
      success: false
    });
  }

  logPreloadEnd(path: string, success: boolean): void {
    const metric = this.preloadMetrics.get(path);
    if (metric) {
      metric.endTime = Date.now();
      metric.success = success;
    }
  }

  getMetrics(): Array<{ path: string; duration: number; success: boolean }> {
    return Array.from(this.preloadMetrics.entries()).map(([path, metric]) => ({
      path,
      duration: (metric.endTime || Date.now()) - metric.startTime,
      success: metric.success
    }));
  }
}
```

### 3. Provide Fallback Loading

```typescript
// Always ensure on-demand loading works even if preloading fails
@Injectable({ providedIn: 'root' })
export class SafePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (!route.data?.['preload']) {
      return of(null);
    }

    return load().pipe(
      tap(() => console.log(`Preloaded: ${route.path}`)),
      catchError(error => {
        console.error(`Failed to preload ${route.path}:`, error);
        return of(null); // Don't break app if preload fails
      })
    );
  }
}
```

## Interview Questions

### Q1: What are preloading strategies in Angular?
**Answer:** Preloading strategies determine when and how lazy-loaded modules are loaded in the background. Angular provides PreloadAllModules and NoPreloading, and supports custom strategies for fine-grained control.

### Q2: When should you use PreloadAllModules?
**Answer:** Use PreloadAllModules for small applications with few lazy modules, when users typically access most routes, and when network conditions are generally good. Avoid for large apps or slow connections.

### Q3: How do you create a custom preloading strategy?
**Answer:** Implement the PreloadingStrategy interface with a preload() method. Return the load() observable to preload, or of(null) to skip. Provide the strategy with withPreloading() in app config.

### Q4: What is network-aware preloading?
**Answer:** Network-aware preloading adjusts loading behavior based on connection speed, data saver mode, and bandwidth. It prevents preloading on slow connections to preserve user bandwidth.

### Q5: How do you preload modules based on priority?
**Answer:** Add priority data to routes and implement a strategy that sorts and preloads high-priority routes first, potentially with delays for lower-priority modules.

### Q6: Can you preload based on user behavior?
**Answer:** Yes, implement strategies that track user activity, hover events, or navigation patterns to predictively preload likely next destinations.

### Q7: What's the difference between preloading and lazy loading?
**Answer:** Lazy loading loads modules on-demand when routes are accessed. Preloading loads modules in the background after initial load, before they're needed, improving subsequent navigation speed.

### Q8: How do you prevent preloading too many modules?
**Answer:** Implement quota-based strategies that track total preloaded size, use selective preloading with explicit flags, or employ network-aware strategies that respect connection limits.

## Key Takeaways

1. **Preloading strategies** optimize lazy module loading
2. **PreloadAllModules** preloads everything after initial load
3. **NoPreloading** loads strictly on-demand
4. **Custom strategies** provide fine-grained control
5. **Network-aware** preloading respects connection speed
6. **Selective preloading** uses route data flags
7. **Priority-based** loading handles important modules first
8. **User behavior** can guide preloading decisions
9. **Monitor performance** to optimize strategy
10. **Always handle errors** gracefully in preloading

## Resources

- [Angular Preloading Documentation](https://angular.io/guide/router#preloading)
- [PreloadingStrategy API](https://angular.io/api/router/PreloadingStrategy)
- [PreloadAllModules API](https://angular.io/api/router/PreloadAllModules)
- [Lazy Loading Guide](https://angular.io/guide/lazy-loading-ngmodules)
- [Network Information API](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API)
- [Angular Performance Guide](https://angular.io/guide/performance-best-practices)
