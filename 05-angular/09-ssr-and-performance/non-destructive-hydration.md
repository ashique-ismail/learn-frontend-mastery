# Non-Destructive Hydration in Angular

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Hydration](#understanding-hydration)
3. [Enabling Hydration](#enabling-hydration)
4. [How Hydration Works](#how-hydration-works)
5. [TransferState API](#transferstate-api)
6. [Incremental Hydration](#incremental-hydration)
7. [Server-Client Data Sync](#server-client-data-sync)
8. [Hydration Performance](#hydration-performance)
9. [Debugging Hydration](#debugging-hydration)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)
13. [Key Takeaways](#key-takeaways)
14. [Resources](#resources)

## Introduction

Non-destructive hydration is a modern Angular feature that allows the framework to reuse server-rendered DOM instead of destroying and recreating it during client-side initialization. This dramatically improves performance by reducing JavaScript execution time, preventing layout shifts, and providing a smoother user experience. This guide covers everything from enabling hydration to advanced patterns for optimal hydration performance.

## Understanding Hydration

### What is Hydration?

```typescript
/**
 * Hydration Process:
 * 
 * Without Hydration (Destructive):
 * 1. Server renders HTML
 * 2. Browser displays HTML (user sees content)
 * 3. JavaScript loads
 * 4. Angular destroys server-rendered DOM
 * 5. Angular recreates DOM from scratch
 * 6. Event listeners attached
 * Result: Flash of content, layout shift, slower
 * 
 * With Hydration (Non-Destructive):
 * 1. Server renders HTML with hydration markers
 * 2. Browser displays HTML (user sees content)
 * 3. JavaScript loads
 * 4. Angular reuses existing DOM
 * 5. Event listeners attached directly
 * 6. State transferred from server
 * Result: Smooth, fast, no layout shift
 */
```

### Benefits of Hydration

```typescript
/**
 * Performance Benefits:
 * 
 * 1. Faster Time to Interactive (TTI)
 *    - No DOM recreation needed
 *    - Reduced JavaScript execution
 *    - Event listeners attached faster
 * 
 * 2. Better Core Web Vitals
 *    - Improved Cumulative Layout Shift (CLS)
 *    - Better First Input Delay (FID)
 *    - Lower Total Blocking Time (TBT)
 * 
 * 3. Reduced Memory Usage
 *    - Reuses existing DOM nodes
 *    - Less garbage collection
 *    - Lower memory footprint
 * 
 * 4. Better User Experience
 *    - No content flashing
 *    - Smoother transitions
 *    - Faster interactivity
 */
```

### Hydration vs No Hydration

```typescript
// Without Hydration
@Component({
  selector: 'app-root',
  template: `
    <h1>{{ title }}</h1>
    <button (click)="onClick()">Click Me</button>
  `
})
export class AppComponent {
  title = 'My App';
  onClick() { console.log('Clicked'); }
}

/**
 * Without Hydration Process:
 * Server: Renders "<h1>My App</h1><button>Click Me</button>"
 * Browser: Shows content
 * Angular loads: Destroys DOM, recreates identical DOM
 * Result: Unnecessary work, potential flash
 * 
 * With Hydration Process:
 * Server: Renders with markers "<h1 ng-server-id='1'>My App</h1>"
 * Browser: Shows content
 * Angular loads: Recognizes markers, reuses DOM, adds listeners
 * Result: Fast, smooth, no flash
 */
```

## Enabling Hydration

### Basic Hydration Setup

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideClientHydration } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideClientHydration() // Enable hydration
  ]
};
```

### Hydration with HTTP Cache

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { 
  provideClientHydration, 
  withHttpTransferCacheOptions 
} from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withFetch()),
    provideClientHydration(
      withHttpTransferCacheOptions({
        includePostRequests: true,
        includeRequestsWithAuthHeaders: false
      })
    )
  ]
};
```

### Selective Hydration

```typescript
// Disable hydration for specific component
@Component({
  selector: 'app-no-hydrate',
  standalone: true,
  template: `<div>This component won't be hydrated</div>`,
  hydrate: false // Disable hydration for this component
})
export class NoHydrateComponent {}

// Disable hydration for specific route
export const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    data: { hydrate: false } // Disable hydration for this route
  }
];
```

## How Hydration Works

### Hydration Markers

```typescript
/**
 * Server-rendered HTML with hydration markers:
 */
/*
<html>
  <body>
    <app-root ng-server-context="ssr">
      <div ng-reflect-title="Hello" jsaction="click:;">
        <h1 ngh="0">Hello World</h1>
        <button ngh="1">Click Me</button>
      </div>
    </app-root>
    
    <script type="text/javascript" id="ng-state">
      window.__NGUNIVERSAL_STATE__ = {
        "app-data": { "users": [...] },
        "http-cache": { ... }
      };
    </script>
  </body>
</html>
*/

/**
 * Hydration Process:
 * 1. Angular finds ng-server-context marker
 * 2. Reads ngh (hydration) attributes
 * 3. Matches server nodes to component tree
 * 4. Attaches event listeners to existing nodes
 * 5. Reads __NGUNIVERSAL_STATE__ for data
 * 6. Removes hydration markers
 * 7. Application ready
 */
```

### DOM Node Matching

```typescript
import { 
  Component, 
  OnInit, 
  inject, 
  Renderer2,
  ElementRef 
} from '@angular/core';

@Component({
  selector: 'app-hydration-demo',
  standalone: true,
  template: `
    <div class="container">
      <h1>{{ title }}</h1>
      <p>{{ description }}</p>
      <button (click)="onClick()">Action</button>
    </div>
  `
})
export class HydrationDemoComponent implements OnInit {
  private elementRef = inject(ElementRef);
  private renderer = inject(Renderer2);
  
  title = 'Hydration Demo';
  description = 'Non-destructive hydration example';

  ngOnInit() {
    // During hydration, this component finds existing DOM
    const element = this.elementRef.nativeElement;
    console.log('Element exists:', element.innerHTML.length > 0);
    
    // Angular reuses this DOM instead of recreating it
    // Event listeners are attached to existing elements
  }

  onClick() {
    console.log('Button clicked - listener was hydrated!');
  }
}
```

### Hydration Lifecycle

```typescript
import { 
  Component, 
  OnInit, 
  AfterViewInit,
  inject,
  PLATFORM_ID,
  afterNextRender
} from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-lifecycle',
  standalone: true,
  template: `
    <div>
      <h1>{{ step }}</h1>
      <p>Platform: {{ platform }}</p>
    </div>
  `
})
export class LifecycleComponent implements OnInit, AfterViewInit {
  private platformId = inject(PLATFORM_ID);
  step = '';
  platform = '';

  constructor() {
    this.step = 'Constructor';
    this.platform = isPlatformBrowser(this.platformId) ? 'Browser' : 'Server';
    console.log(`${this.platform}: Constructor`);
    
    // After hydration completes (browser only)
    afterNextRender(() => {
      console.log('Browser: After hydration');
      this.step = 'Hydrated and interactive';
    });
  }

  ngOnInit() {
    console.log(`${this.platform}: ngOnInit`);
    this.step = 'Initialized';
  }

  ngAfterViewInit() {
    console.log(`${this.platform}: ngAfterViewInit`);
    
    if (isPlatformBrowser(this.platformId)) {
      // View is ready and hydrated
      this.step = 'View ready';
    }
  }
}

/**
 * Timeline:
 * Server: Constructor -> ngOnInit -> ngAfterViewInit -> Render HTML
 * Browser: Parse HTML -> Load JS -> Constructor -> ngOnInit -> 
 *          Hydrate DOM -> ngAfterViewInit -> afterNextRender
 */
```

## TransferState API

### Basic TransferState with Hydration

```typescript
import { Component, OnInit, inject } from '@angular/core';
import { TransferState, makeStateKey } from '@angular/platform-browser';
import { HttpClient } from '@angular/common/http';

const DATA_KEY = makeStateKey<any>('api-data');

@Component({
  selector: 'app-transfer-state',
  standalone: true,
  template: `
    <div *ngIf="data">
      <h2>{{ data.title }}</h2>
      <p>{{ data.content }}</p>
    </div>
  `
})
export class TransferStateComponent implements OnInit {
  private transferState = inject(TransferState);
  private http = inject(HttpClient);
  
  data: any;

  ngOnInit() {
    // Try to get data from transfer state
    const cached = this.transferState.get(DATA_KEY, null);
    
    if (cached) {
      // Use cached data (hydration)
      console.log('Using transferred data');
      this.data = cached;
      
      // Clean up to free memory
      this.transferState.remove(DATA_KEY);
    } else {
      // Fetch data (initial server render or client without SSR)
      console.log('Fetching data');
      this.http.get('/api/data').subscribe(response => {
        this.data = response;
        // Will be automatically transferred during SSR
      });
    }
  }
}
```

### Automatic HTTP Transfer Cache

```typescript
// app.config.ts - Enable automatic HTTP caching
import { ApplicationConfig } from '@angular/core';
import { 
  provideClientHydration, 
  withHttpTransferCacheOptions 
} from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withFetch()),
    provideClientHydration(
      withHttpTransferCacheOptions({
        // Include POST requests in cache
        includePostRequests: true,
        
        // Don't cache requests with authorization headers
        includeRequestsWithAuthHeaders: false,
        
        // Only cache specific request headers
        includeHeaders: ['Accept', 'Content-Type'],
        
        // Filter which requests to cache
        filter: (req) => {
          // Only cache GET requests to specific API
          return req.method === 'GET' && 
                 req.url.startsWith('/api/');
        }
      })
    )
  ]
};

// Now HTTP requests are automatically cached
@Component({
  selector: 'app-auto-cache',
  standalone: true,
  template: `
    <div *ngFor="let item of items$ | async">
      {{ item.name }}
    </div>
  `
})
export class AutoCacheComponent {
  private http = inject(HttpClient);
  
  // This request is automatically cached during SSR
  // and transferred to the client
  items$ = this.http.get<any[]>('/api/items');
}
```

### Custom TransferState Helper

```typescript
import { Injectable, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { TransferState, makeStateKey } from '@angular/platform-browser';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class TransferStateHelper {
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  /**
   * Get data from transfer state or fetch it
   */
  getOrSet<T>(
    key: string,
    fetch: () => Observable<T>
  ): Observable<T> {
    const stateKey = makeStateKey<T>(key);
    
    // Check if data exists in transfer state
    if (this.transferState.hasKey(stateKey)) {
      const cached = this.transferState.get(stateKey, null as any);
      this.transferState.remove(stateKey); // Clean up
      return of(cached);
    }
    
    // Fetch data and store in transfer state if on server
    return fetch().pipe(
      tap(data => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(stateKey, data);
        }
      })
    );
  }

  /**
   * Set value in transfer state (server only)
   */
  set<T>(key: string, value: T): void {
    if (isPlatformServer(this.platformId)) {
      const stateKey = makeStateKey<T>(key);
      this.transferState.set(stateKey, value);
    }
  }

  /**
   * Get value from transfer state (client only)
   */
  get<T>(key: string, defaultValue: T): T {
    const stateKey = makeStateKey<T>(key);
    const value = this.transferState.get(stateKey, defaultValue);
    this.transferState.remove(stateKey); // Clean up
    return value;
  }

  /**
   * Check if key exists
   */
  has(key: string): boolean {
    const stateKey = makeStateKey(key);
    return this.transferState.hasKey(stateKey);
  }
}

// Usage
@Component({
  selector: 'app-users',
  standalone: true,
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UsersComponent {
  private http = inject(HttpClient);
  private transferHelper = inject(TransferStateHelper);

  users$ = this.transferHelper.getOrSet(
    'users',
    () => this.http.get<any[]>('/api/users')
  );
}
```

## Incremental Hydration

### Lazy Hydration Strategy

```typescript
import { 
  Component, 
  ViewChild, 
  ElementRef, 
  AfterViewInit,
  inject,
  PLATFORM_ID
} from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-lazy-hydrate',
  standalone: true,
  template: `
    <div>
      <h1>Above the fold - hydrated immediately</h1>
      
      <div #lazySection class="below-fold">
        <h2>Below the fold - hydrated when visible</h2>
        <app-heavy-component></app-heavy-component>
      </div>
    </div>
  `
})
export class LazyHydrateComponent implements AfterViewInit {
  @ViewChild('lazySection') lazySection!: ElementRef;
  private platformId = inject(PLATFORM_ID);

  ngAfterViewInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Use Intersection Observer for lazy hydration
      const observer = new IntersectionObserver((entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            this.hydrateSection();
            observer.disconnect();
          }
        });
      });

      observer.observe(this.lazySection.nativeElement);
    }
  }

  private hydrateSection() {
    console.log('Hydrating below-the-fold content');
    // Component hydration happens automatically
  }
}
```

### Viewport-Based Hydration

```typescript
import { Directive, ElementRef, OnInit, inject } from '@angular/core';

@Directive({
  selector: '[appHydrateOnVisible]',
  standalone: true
})
export class HydrateOnVisibleDirective implements OnInit {
  private elementRef = inject(ElementRef);

  ngOnInit() {
    if (typeof IntersectionObserver !== 'undefined') {
      const observer = new IntersectionObserver(
        (entries) => {
          entries.forEach(entry => {
            if (entry.isIntersecting) {
              this.hydrate();
              observer.disconnect();
            }
          });
        },
        { rootMargin: '50px' } // Start hydrating 50px before visible
      );

      observer.observe(this.elementRef.nativeElement);
    } else {
      // Fallback: hydrate immediately
      this.hydrate();
    }
  }

  private hydrate() {
    console.log('Hydrating component');
    this.elementRef.nativeElement.setAttribute('data-hydrated', 'true');
  }
}

// Usage
@Component({
  selector: 'app-gallery',
  standalone: true,
  imports: [HydrateOnVisibleDirective],
  template: `
    <div appHydrateOnVisible>
      <app-image-grid [images]="images"></app-image-grid>
    </div>
  `
})
export class GalleryComponent {
  images = [...]; // Large image dataset
}
```

### Interaction-Based Hydration

```typescript
import { 
  Component, 
  ViewChild, 
  ElementRef, 
  inject,
  afterNextRender 
} from '@angular/core';

@Component({
  selector: 'app-modal-trigger',
  standalone: true,
  template: `
    <button #trigger (click)="openModal()">
      Open Modal
    </button>
    
    <div *ngIf="isModalHydrated">
      <app-modal></app-modal>
    </div>
  `
})
export class ModalTriggerComponent {
  @ViewChild('trigger') trigger!: ElementRef;
  isModalHydrated = false;

  constructor() {
    afterNextRender(() => {
      // Hydrate modal component only when user interacts
      this.trigger.nativeElement.addEventListener('click', () => {
        this.isModalHydrated = true;
      }, { once: true });
    });
  }

  openModal() {
    console.log('Modal opening - now hydrated');
  }
}
```

## Server-Client Data Sync

### Synchronizing Component State

```typescript
import { Component, OnInit, inject } from '@angular/core';
import { TransferState, makeStateKey } from '@angular/platform-browser';

const COUNTER_KEY = makeStateKey<number>('counter');

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <div>
      <p>Count: {{ count }}</p>
      <button (click)="increment()">Increment</button>
    </div>
  `
})
export class CounterComponent implements OnInit {
  private transferState = inject(TransferState);
  count = 0;

  ngOnInit() {
    // Get initial state from server
    this.count = this.transferState.get(COUNTER_KEY, 0);
    this.transferState.remove(COUNTER_KEY);
  }

  increment() {
    this.count++;
  }

  // On server, save state before rendering
  ngOnDestroy() {
    if (typeof window === 'undefined') { // Server only
      this.transferState.set(COUNTER_KEY, this.count);
    }
  }
}
```

### Form State Transfer

```typescript
import { Component, OnInit, inject } from '@angular/core';
import { FormBuilder, FormGroup, ReactiveFormsModule } from '@angular/forms';
import { TransferState, makeStateKey } from '@angular/platform-browser';

const FORM_STATE_KEY = makeStateKey<any>('form-state');

@Component({
  selector: 'app-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form">
      <input formControlName="name" placeholder="Name">
      <input formControlName="email" placeholder="Email">
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent implements OnInit {
  private fb = inject(FormBuilder);
  private transferState = inject(TransferState);
  
  form!: FormGroup;

  ngOnInit() {
    // Create form
    this.form = this.fb.group({
      name: [''],
      email: ['']
    });

    // Restore form state from transfer state
    const savedState = this.transferState.get(FORM_STATE_KEY, null);
    if (savedState) {
      this.form.patchValue(savedState);
      this.transferState.remove(FORM_STATE_KEY);
    }

    // Save form state on changes (for server)
    this.form.valueChanges.subscribe(value => {
      if (typeof window === 'undefined') {
        this.transferState.set(FORM_STATE_KEY, value);
      }
    });
  }
}
```

### Observable State Sync

```typescript
import { Injectable, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { BehaviorSubject, Observable } from 'rxjs';
import { TransferState, makeStateKey } from '@angular/platform-browser';

@Injectable({ providedIn: 'root' })
export class StateService {
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);
  
  private userSubject = new BehaviorSubject<any>(null);
  user$ = this.userSubject.asObservable();

  constructor() {
    // Restore state on client
    const USER_KEY = makeStateKey<any>('user-state');
    const savedUser = this.transferState.get(USER_KEY, null);
    
    if (savedUser) {
      this.userSubject.next(savedUser);
      this.transferState.remove(USER_KEY);
    }
  }

  setUser(user: any) {
    this.userSubject.next(user);
    
    // Save state on server
    if (isPlatformServer(this.platformId)) {
      const USER_KEY = makeStateKey<any>('user-state');
      this.transferState.set(USER_KEY, user);
    }
  }

  getUser(): Observable<any> {
    return this.user$;
  }
}
```

## Hydration Performance

### Measuring Hydration Performance

```typescript
import { Component, OnInit, afterNextRender } from '@angular/core';

@Component({
  selector: 'app-perf',
  standalone: true,
  template: `<div>Performance Demo</div>`
})
export class PerfComponent implements OnInit {
  constructor() {
    // Measure hydration time
    afterNextRender(() => {
      if (performance.getEntriesByName('hydration-start').length) {
        performance.mark('hydration-end');
        performance.measure(
          'hydration',
          'hydration-start',
          'hydration-end'
        );
        
        const measure = performance.getEntriesByName('hydration')[0];
        console.log(`Hydration took ${measure.duration}ms`);
      }
    });
  }

  ngOnInit() {
    performance.mark('hydration-start');
  }
}
```

### Optimizing Hydration Performance

```typescript
/**
 * Hydration Performance Tips:
 * 
 * 1. Minimize Server-Rendered Size
 *    - Remove unnecessary whitespace
 *    - Compress HTML response
 *    - Minimize inline styles
 * 
 * 2. Reduce JavaScript Bundle Size
 *    - Code split routes
 *    - Lazy load components
 *    - Remove unused dependencies
 * 
 * 3. Prioritize Critical Content
 *    - Hydrate above-the-fold first
 *    - Defer below-the-fold hydration
 *    - Use interaction-based hydration
 * 
 * 4. Optimize Data Transfer
 *    - Only transfer necessary data
 *    - Compress transfer state data
 *    - Clean up transfer state after use
 * 
 * 5. Use Proper Caching
 *    - Cache static assets aggressively
 *    - Use HTTP transfer cache
 *    - Implement service worker caching
 */

// Example: Optimized component
@Component({
  selector: 'app-optimized',
  standalone: true,
  template: `
    <!-- Critical content -->
    <header>{{ title }}</header>
    
    <!-- Lazy-loaded, deferred hydration -->
    <div #belowFold>
      <app-heavy-component *ngIf="hydrateHeavy"></app-heavy-component>
    </div>
  `
})
export class OptimizedComponent implements AfterViewInit {
  @ViewChild('belowFold') belowFold!: ElementRef;
  title = 'My App';
  hydrateHeavy = false;

  ngAfterViewInit() {
    // Defer heavy component hydration
    requestIdleCallback(() => {
      this.hydrateHeavy = true;
    });
  }
}
```

## Debugging Hydration

### Hydration Debugging Tools

```typescript
// Enable hydration debugging
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "development": {
              "optimization": false,
              "hydration": true,
              "verbose": true // Enable verbose hydration logging
            }
          }
        }
      }
    }
  }
}

// Check hydration in component
import { Component, OnInit, inject, ɵENABLE_DEBUG_TOOLS } from '@angular/core';

@Component({
  selector: 'app-debug',
  standalone: true,
  template: `<div>Debug</div>`
})
export class DebugComponent implements OnInit {
  ngOnInit() {
    // Check if hydration is enabled
    console.log('Hydration enabled:', 
      document.body.hasAttribute('ng-server-context')
    );
    
    // Check transfer state
    const stateScript = document.getElementById('ng-state');
    if (stateScript) {
      console.log('Transfer state exists');
    }
  }
}
```

### Common Hydration Issues

```typescript
/**
 * Hydration Mismatch Errors:
 * 
 * Error: "NG0500: During hydration Angular expected <div> but found <span>"
 * Cause: Server and client rendered different HTML
 * Solution: Ensure consistent rendering on server and client
 */

// BAD: Different output on server/client
@Component({
  template: `<div>{{ getTime() }}</div>`
})
export class BadComponent {
  getTime() {
    // Different value on server and client!
    return Date.now();
  }
}

// GOOD: Consistent output
@Component({
  template: `<div>{{ time }}</div>`
})
export class GoodComponent implements OnInit {
  time: number = 0;

  ngOnInit() {
    // Set once, same on server and client
    this.time = Date.now();
  }
}
```

## Common Mistakes

### 1. Accessing Browser APIs During Hydration

```typescript
// BAD: Will cause hydration errors
@Component({ template: '' })
export class BadComponent implements OnInit {
  ngOnInit() {
    const width = window.innerWidth; // Error during SSR
  }
}

// GOOD: Check platform
@Component({ template: '' })
export class GoodComponent implements OnInit {
  constructor() {
    afterNextRender(() => {
      const width = window.innerWidth; // Safe
    });
  }
}
```

### 2. Not Cleaning Up Transfer State

```typescript
// BAD: Memory leak
@Component({ template: '' })
export class BadComponent {
  ngOnInit() {
    const data = this.transferState.get(KEY, null);
    // Never removed!
  }
}

// GOOD: Clean up
@Component({ template: '' })
export class GoodComponent {
  ngOnInit() {
    const data = this.transferState.get(KEY, null);
    this.transferState.remove(KEY); // Free memory
  }
}
```

### 3. Inconsistent Rendering

```typescript
// BAD: Random content
@Component({
  template: `<div>{{ Math.random() }}</div>`
})
export class BadComponent {} // Different on server/client!

// GOOD: Consistent content
@Component({
  template: `<div>{{ randomValue }}</div>`
})
export class GoodComponent implements OnInit {
  randomValue = 0;
  
  ngOnInit() {
    this.randomValue = Math.random(); // Same value used for hydration
  }
}
```

## Best Practices

### 1. Enable HTTP Transfer Cache

```typescript
// Always enable for better performance
provideClientHydration(
  withHttpTransferCacheOptions({
    includePostRequests: true
  })
)
```

### 2. Clean Up Transfer State

```typescript
// Always remove after use
const data = this.transferState.get(KEY, null);
if (data) {
  this.data = data;
  this.transferState.remove(KEY); // Important!
}
```

### 3. Use afterNextRender for Browser Code

```typescript
constructor() {
  afterNextRender(() => {
    // Guaranteed to run after hydration
    // Safe for browser APIs
  });
}
```

### 4. Measure Hydration Performance

```typescript
// Track hydration metrics
afterNextRender(() => {
  const hydrationTime = performance.getEntriesByName('hydration')[0];
  console.log('Hydration time:', hydrationTime.duration);
});
```

## Interview Questions

### Q1: What is non-destructive hydration?

**Answer:** Non-destructive hydration is when Angular reuses server-rendered DOM instead of destroying and recreating it. This improves performance by reducing JavaScript execution, preventing layout shifts, and making the app interactive faster.

### Q2: How does Angular know which DOM nodes to reuse?

**Answer:** Angular adds hydration markers (ngh attributes) to server-rendered HTML. During hydration, Angular matches these markers to components and reuses the existing DOM nodes instead of creating new ones.

### Q3: What's the difference between hydration and rehydration?

**Answer:** These terms are often used interchangeably. Both refer to the process of making server-rendered HTML interactive on the client. "Non-destructive hydration" specifically emphasizes that the DOM isn't destroyed.

### Q4: When should you disable hydration?

**Answer:** Disable hydration for components that intentionally render differently on server and client, heavily interactive components that don't benefit from SSR, or when debugging hydration mismatch errors.

### Q5: How does TransferState work with hydration?

**Answer:** TransferState serializes data on the server and includes it in the HTML. During hydration, Angular reads this data from the HTML, preventing duplicate HTTP requests and ensuring consistency.

### Q6: What causes hydration mismatch errors?

**Answer:** Hydration mismatches occur when server and client render different HTML. Common causes include Date.now(), Math.random(), browser-only APIs, or conditional rendering based on platform.

### Q7: How do you debug hydration issues?

**Answer:** Enable verbose logging, check for hydration markers in HTML, verify transfer state, ensure consistent rendering, and use Angular DevTools to inspect the component tree.

### Q8: Can you partially hydrate a page?

**Answer:** Yes, you can use lazy hydration strategies to hydrate components based on viewport visibility, user interaction, or time delays. This improves initial load performance.

## Key Takeaways

1. **Non-destructive hydration reuses server-rendered DOM** instead of recreating it
2. **Enable hydration with provideClientHydration** in app.config
3. **Use HTTP transfer cache** to prevent duplicate requests
4. **Clean up transfer state** after reading to free memory
5. **Ensure consistent rendering** on server and client to avoid mismatches
6. **Use afterNextRender** for guaranteed post-hydration browser code
7. **Consider lazy hydration** for below-the-fold content
8. **Monitor hydration performance** with performance marks
9. **Hydration markers are automatically added** by Angular during SSR
10. **Hydration improves Core Web Vitals** especially CLS and TTI

## Resources

### Official Documentation
- [Angular Hydration Guide](https://angular.dev/guide/hydration)
- [TransferState API](https://angular.dev/api/platform-browser/TransferState)
- [provideClientHydration](https://angular.dev/api/platform-browser/provideClientHydration)

### Articles
- "Understanding Angular Hydration" - Angular Blog
- "Non-Destructive Hydration Deep Dive" - Web.dev
- "Optimizing Hydration Performance" - Angular University

### Video Tutorials
- "Angular Hydration Explained" - ng-conf
- "Mastering Hydration" - Angular Connect
- "SSR Performance with Hydration" - Google Chrome Developers

### Tools
- Angular DevTools - Hydration debugging
- Chrome DevTools - Performance profiling
- Lighthouse - Measure hydration impact on Core Web Vitals
