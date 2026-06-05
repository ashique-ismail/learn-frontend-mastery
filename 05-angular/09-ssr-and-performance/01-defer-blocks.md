# Defer Blocks in Angular

## The Idea

**In plain English:** A defer block is a way to tell your app "don't download and show this part of the page yet — wait until the right moment." This keeps the app fast on first load by only fetching the code you actually need right now, and postponing the rest.

**Real-world analogy:** Imagine a buffet restaurant that keeps most dishes in the kitchen and only brings them out when a guest walks up to that section of the buffet line, instead of putting every single dish on the table the moment the restaurant opens.

- The dishes in the kitchen = components whose code has not been downloaded yet
- A guest walking up to a section = the trigger (scrolling into view, clicking a button, the browser going idle)
- The dish arriving at the table = the deferred component loading and appearing on screen

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding Defer Blocks](#understanding-defer-blocks)
3. [Basic @defer Syntax](#basic-defer-syntax)
4. [Defer Triggers](#defer-triggers)
5. [Prefetch Strategies](#prefetch-strategies)
6. [Loading and Error States](#loading-and-error-states)
7. [Advanced Defer Patterns](#advanced-defer-patterns)
8. [Performance Optimization](#performance-optimization)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)
12. [Key Takeaways](#key-takeaways)
13. [Resources](#resources)

## Introduction

Defer blocks (`@defer`) are a powerful Angular feature that enables declarative lazy loading of components, directives, and pipes directly in templates. Instead of programmatically loading code, you can use the `@defer` syntax to specify when and how content should be loaded. This dramatically simplifies code splitting, improves initial load performance, and provides built-in loading and error states.

## Understanding Defer Blocks

### Why Use Defer Blocks?

```typescript
/**
 * Benefits of @defer:
 * 
 * 1. Reduced Initial Bundle Size
 *    - Components loaded only when needed
 *    - Smaller main bundle
 *    - Faster initial page load
 * 
 * 2. Declarative Lazy Loading
 *    - No programmatic loading code
 *    - Clear intent in template
 *    - Easier to maintain
 * 
 * 3. Built-in Loading States
 *    - Loading placeholder
 *    - Error handling
 *    - Minimum loading time
 * 
 * 4. Flexible Triggers
 *    - Idle time
 *    - Viewport visibility
 *    - User interaction
 *    - Timer-based
 *    - Immediate
 * 
 * 5. Prefetch Support
 *    - Load code before showing
 *    - Better user experience
 *    - Configurable prefetch timing
 */
```

### Before and After @defer

```typescript
// Before @defer - Manual lazy loading
@Component({
  selector: 'app-old-way',
  standalone: true,
  template: `
    <button (click)="showHeavy = true">Show Component</button>
    
    @if (showHeavy) {
      <app-heavy></app-heavy>
    }
  `
})
export class OldWayComponent {
  showHeavy = false;
  
  // Heavy component is included in main bundle
  // even if never shown
}

// After @defer - Declarative lazy loading
@Component({
  selector: 'app-new-way',
  standalone: true,
  template: `
    <button (click)="showHeavy = true">Show Component</button>
    
    @defer (when showHeavy) {
      <app-heavy></app-heavy>
    }
    
    @loading {
      <div>Loading...</div>
    }
    
    @error {
      <div>Failed to load</div>
    }
  `
})
export class NewWayComponent {
  showHeavy = false;
  
  // Heavy component code split automatically
  // Loaded only when showHeavy becomes true
}
```

## Basic @defer Syntax

### Simple Defer Block

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-simple-defer',
  standalone: true,
  template: `
    <h1>Above the fold content</h1>
    
    @defer {
      <app-below-fold></app-below-fold>
    }
    
    @loading {
      <div class="skeleton">Loading...</div>
    }
    
    @error {
      <div class="error">Failed to load content</div>
    }
    
    @placeholder {
      <div class="placeholder">Placeholder content</div>
    }
  `
})
export class SimpleDeferComponent {}
```

### Defer Block States

```typescript
@Component({
  selector: 'app-defer-states',
  standalone: true,
  template: `
    @defer {
      <!-- Main content - shown after loading -->
      <app-heavy-component></app-heavy-component>
    }
    
    @placeholder {
      <!-- Shown before defer condition is met -->
      <div>Click below to load</div>
    }
    
    @loading (minimum 500ms) {
      <!-- Shown while loading, minimum 500ms -->
      <div class="spinner">Loading...</div>
    }
    
    @error {
      <!-- Shown if loading fails -->
      <div class="error">
        <p>Something went wrong</p>
        <button (click)="retry()">Retry</button>
      </div>
    }
  `,
  styles: [`
    .spinner { animation: spin 1s infinite linear; }
    .error { color: red; padding: 16px; }
  `]
})
export class DeferStatesComponent {
  retry() {
    // Retry logic
    window.location.reload();
  }
}
```

## Defer Triggers

### Idle Trigger

```typescript
@Component({
  selector: 'app-idle-defer',
  standalone: true,
  template: `
    <h1>Main Content</h1>
    
    <!-- Load when browser is idle -->
    @defer (on idle) {
      <app-analytics-dashboard></app-analytics-dashboard>
    }
    
    @loading {
      <div>Loading dashboard...</div>
    }
  `
})
export class IdleDeferComponent {}
```

### Viewport Trigger

```typescript
@Component({
  selector: 'app-viewport-defer',
  standalone: true,
  template: `
    <div class="above-fold">
      <h1>Top Content</h1>
      <p>Visible immediately</p>
    </div>
    
    <div class="spacer" style="height: 200vh;"></div>
    
    <!-- Load when placeholder enters viewport -->
    @defer (on viewport) {
      <app-comments></app-comments>
    }
    
    @placeholder {
      <div class="placeholder">
        Scroll down to load comments
      </div>
    }
    
    @loading {
      <div>Loading comments...</div>
    }
  `
})
export class ViewportDeferComponent {}
```

### Viewport with Reference

```typescript
@Component({
  selector: 'app-viewport-ref',
  standalone: true,
  template: `
    <h1>Article</h1>
    <p>Content...</p>
    
    <!-- Trigger element -->
    <div #loadTrigger class="load-trigger"></div>
    
    <!-- Load when trigger enters viewport -->
    @defer (on viewport(loadTrigger)) {
      <app-related-articles></app-related-articles>
    }
    
    @placeholder {
      <div>More articles will load here</div>
    }
  `
})
export class ViewportRefComponent {}
```

### Interaction Triggers

```typescript
@Component({
  selector: 'app-interaction-defer',
  standalone: true,
  template: `
    <!-- Load on click -->
    <button #loadBtn>Load Content</button>
    
    @defer (on interaction(loadBtn)) {
      <app-modal-content></app-modal-content>
    }
    
    @loading {
      <div>Opening modal...</div>
    }
    
    <!-- Load on hover -->
    <div #hoverZone class="hover-zone">
      Hover to preview
    </div>
    
    @defer (on hover(hoverZone)) {
      <app-preview></app-preview>
    }
    
    <!-- Load on focus -->
    <input #searchInput placeholder="Search...">
    
    @defer (on focus(searchInput)) {
      <app-search-results></app-search-results>
    }
  `
})
export class InteractionDeferComponent {}
```

### Timer Trigger

```typescript
@Component({
  selector: 'app-timer-defer',
  standalone: true,
  template: `
    <h1>Welcome!</h1>
    
    <!-- Load after 5 seconds -->
    @defer (on timer(5s)) {
      <app-promotional-banner></app-promotional-banner>
    }
    
    @placeholder {
      <div>Banner will appear in 5 seconds</div>
    }
    
    <!-- Load after 2 seconds, show for minimum 1 second -->
    @defer (on timer(2000ms)) {
      <app-notification></app-notification>
    }
    
    @loading (minimum 1s) {
      <div>Preparing notification...</div>
    }
  `
})
export class TimerDeferComponent {}
```

### Immediate Trigger

```typescript
@Component({
  selector: 'app-immediate-defer',
  standalone: true,
  template: `
    <!-- Load immediately, but in separate chunk -->
    @defer (on immediate) {
      <app-important-but-heavy></app-important-but-heavy>
    }
    
    @loading {
      <div>Loading...</div>
    }
  `
})
export class ImmediateDeferComponent {}
```

### When Trigger (Conditional)

```typescript
@Component({
  selector: 'app-when-defer',
  standalone: true,
  template: `
    <button (click)="showContent = true">Show Content</button>
    
    <!-- Load when condition becomes true -->
    @defer (when showContent) {
      <app-content></app-content>
    }
    
    @placeholder {
      <div>Click button to load</div>
    }
    
    <!-- Load when user is authenticated -->
    @defer (when isAuthenticated$ | async) {
      <app-user-dashboard></app-user-dashboard>
    }
    
    @placeholder {
      <div>Please log in</div>
    }
  `
})
export class WhenDeferComponent {
  showContent = false;
  isAuthenticated$ = this.authService.isAuthenticated$;
  
  constructor(private authService: AuthService) {}
}
```

### Multiple Triggers

```typescript
@Component({
  selector: 'app-multiple-triggers',
  standalone: true,
  template: `
    <div #viewport class="viewport-trigger"></div>
    <button #clickBtn>Load</button>
    
    <!-- Load on any trigger: viewport OR click OR after 5s -->
    @defer (on viewport(viewport); on interaction(clickBtn); on timer(5s)) {
      <app-content></app-content>
    }
    
    @loading {
      <div>Loading content...</div>
    }
  `
})
export class MultipleTriggersComponent {}
```

## Prefetch Strategies

### Prefetch on Idle

```typescript
@Component({
  selector: 'app-prefetch-idle',
  standalone: true,
  template: `
    <button #showBtn>Show Modal</button>
    
    <!-- Prefetch when idle, show on click -->
    @defer (on interaction(showBtn); prefetch on idle) {
      <app-modal></app-modal>
    }
    
    @loading {
      <div>Opening modal...</div>
    }
  `
})
export class PrefetchIdleComponent {}
```

### Prefetch on Viewport

```typescript
@Component({
  selector: 'app-prefetch-viewport',
  standalone: true,
  template: `
    <div #trigger class="trigger"></div>
    
    <!-- Prefetch when trigger near viewport, show when in viewport -->
    @defer (
      on viewport(trigger); 
      prefetch on viewport(trigger)
    ) {
      <app-image-gallery></app-image-gallery>
    }
    
    @loading {
      <div>Loading gallery...</div>
    }
  `
})
export class PrefetchViewportComponent {}
```

### Prefetch on Hover

```typescript
@Component({
  selector: 'app-prefetch-hover',
  standalone: true,
  template: `
    <button #previewBtn>Preview</button>
    
    <!-- Prefetch on hover, show on click -->
    @defer (
      on interaction(previewBtn); 
      prefetch on hover(previewBtn)
    ) {
      <app-preview-panel></app-preview-panel>
    }
    
    @loading (minimum 300ms) {
      <div class="loading-spinner"></div>
    }
  `
})
export class PrefetchHoverComponent {}
```

### Prefetch with Timer

```typescript
@Component({
  selector: 'app-prefetch-timer',
  standalone: true,
  template: `
    <button #loadBtn>Load Content</button>
    
    <!-- Prefetch after 2s, show on click -->
    @defer (
      on interaction(loadBtn); 
      prefetch on timer(2s)
    ) {
      <app-heavy-content></app-heavy-content>
    }
    
    @loading {
      <div>Loading...</div>
    }
  `
})
export class PrefetchTimerComponent {}
```

### Prefetch Immediately

```typescript
@Component({
  selector: 'app-prefetch-immediate',
  standalone: true,
  template: `
    <!-- Prefetch immediately, show on condition -->
    @defer (
      when isReady; 
      prefetch on immediate
    ) {
      <app-dashboard></app-dashboard>
    }
    
    @placeholder {
      <div>Preparing dashboard...</div>
    }
  `
})
export class PrefetchImmediateComponent {
  isReady = false;
  
  ngOnInit() {
    setTimeout(() => this.isReady = true, 1000);
  }
}
```

## Loading and Error States

### Minimum Loading Duration

```typescript
@Component({
  selector: 'app-minimum-loading',
  standalone: true,
  template: `
    @defer (on idle) {
      <app-content></app-content>
    }
    
    <!-- Show loading for at least 500ms to avoid flash -->
    @loading (minimum 500ms) {
      <div class="skeleton-loader">
        <div class="skeleton-line"></div>
        <div class="skeleton-line"></div>
        <div class="skeleton-line"></div>
      </div>
    }
  `,
  styles: [`
    .skeleton-line {
      height: 20px;
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 200% 100%;
      animation: loading 1.5s infinite;
      margin: 8px 0;
    }
    
    @keyframes loading {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class MinimumLoadingComponent {}
```

### After Loading Duration

```typescript
@Component({
  selector: 'app-after-loading',
  standalone: true,
  template: `
    @defer (on viewport) {
      <app-content></app-content>
    }
    
    <!-- Show loading only if it takes more than 100ms -->
    @loading (after 100ms; minimum 500ms) {
      <div class="spinner">Loading...</div>
    }
    
    @placeholder {
      <div class="placeholder-content"></div>
    }
  `
})
export class AfterLoadingComponent {}
```

### Error Recovery

```typescript
@Component({
  selector: 'app-error-recovery',
  standalone: true,
  template: `
    @defer (when shouldLoad) {
      <app-content></app-content>
    }
    
    @loading {
      <div class="loading">
        <div class="spinner"></div>
        <p>Loading content...</p>
      </div>
    }
    
    @error {
      <div class="error-container">
        <div class="error-icon">⚠️</div>
        <h3>Failed to load content</h3>
        <p>{{ errorMessage }}</p>
        <button (click)="retry()" class="retry-btn">
          Retry
        </button>
        <button (click)="shouldLoad = false" class="cancel-btn">
          Cancel
        </button>
      </div>
    }
  `,
  styles: [`
    .error-container {
      padding: 24px;
      text-align: center;
      background: #fff3f3;
      border: 1px solid #ffcdd2;
      border-radius: 8px;
    }
    .error-icon { font-size: 48px; }
    .retry-btn {
      background: #2196F3;
      color: white;
      padding: 8px 16px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
  `]
})
export class ErrorRecoveryComponent {
  shouldLoad = false;
  errorMessage = 'Network error occurred';
  
  retry() {
    this.shouldLoad = false;
    setTimeout(() => this.shouldLoad = true, 100);
  }
}
```

## Advanced Defer Patterns

### Nested Defer Blocks

```typescript
@Component({
  selector: 'app-nested-defer',
  standalone: true,
  template: `
    @defer (on idle) {
      <div class="main-content">
        <h2>Main Content</h2>
        
        @defer (on viewport) {
          <div class="sub-content">
            <h3>Sub Content</h3>
            
            @defer (on interaction) {
              <app-detailed-view></app-detailed-view>
            }
            
            @placeholder {
              <button>Load Details</button>
            }
          </div>
        }
        
        @placeholder {
          <div>Scroll to load sub-content</div>
        }
      </div>
    }
    
    @loading {
      <div>Loading main content...</div>
    }
  `
})
export class NestedDeferComponent {}
```

### Conditional Defer with Fallback

```typescript
@Component({
  selector: 'app-conditional-defer',
  standalone: true,
  template: `
    @if (isPremiumUser) {
      @defer (on idle) {
        <app-premium-features></app-premium-features>
      }
      
      @loading {
        <div>Loading premium features...</div>
      }
    } @else {
      <app-upgrade-prompt></app-upgrade-prompt>
    }
  `
})
export class ConditionalDeferComponent {
  isPremiumUser = false;
}
```

### Dynamic Defer with Signals

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-signal-defer',
  standalone: true,
  template: `
    <button (click)="tabIndex.set(0)">Tab 1</button>
    <button (click)="tabIndex.set(1)">Tab 2</button>
    <button (click)="tabIndex.set(2)">Tab 3</button>
    
    @defer (when tabIndex() === 0) {
      <app-tab-one></app-tab-one>
    }
    
    @defer (when tabIndex() === 1) {
      <app-tab-two></app-tab-two>
    }
    
    @defer (when tabIndex() === 2) {
      <app-tab-three></app-tab-three>
    }
    
    @loading {
      <div>Loading tab content...</div>
    }
  `
})
export class SignalDeferComponent {
  tabIndex = signal(0);
}
```

### Defer with Route Data

```typescript
import { Component } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-route-defer',
  standalone: true,
  template: `
    @defer (when showAdminPanel$ | async) {
      <app-admin-panel></app-admin-panel>
    }
    
    @defer (when showUserDashboard$ | async) {
      <app-user-dashboard></app-user-dashboard>
    }
    
    @placeholder {
      <div>Loading appropriate view...</div>
    }
  `
})
export class RouteDeferComponent {
  showAdminPanel$ = this.route.data.pipe(
    map(data => data['role'] === 'admin')
  );
  
  showUserDashboard$ = this.route.data.pipe(
    map(data => data['role'] === 'user')
  );
  
  constructor(private route: ActivatedRoute) {}
}
```

## Performance Optimization

### Optimizing Large Components

```typescript
@Component({
  selector: 'app-optimized-defer',
  standalone: true,
  template: `
    <!-- Critical content in main bundle -->
    <header>
      <app-navigation></app-navigation>
    </header>
    
    <main>
      <app-hero-section></app-hero-section>
      
      <!-- Defer heavy components -->
      @defer (on viewport; prefetch on idle) {
        <app-feature-grid></app-feature-grid>
      }
      
      @defer (on viewport; prefetch on idle) {
        <app-testimonials></app-testimonials>
      }
      
      @defer (on viewport) {
        <app-pricing-table></app-pricing-table>
      }
      
      @defer (on viewport) {
        <app-contact-form></app-contact-form>
      }
    </main>
    
    <!-- Footer loaded on idle -->
    @defer (on idle) {
      <footer>
        <app-footer></app-footer>
      </footer>
    }
  `
})
export class OptimizedDeferComponent {}
```

### Bundle Size Analysis

```typescript
/**
 * Measuring Defer Impact:
 * 
 * Before @defer:
 * - main.js: 500 KB
 * - Total: 500 KB
 * - Initial load: 2.5s
 * 
 * After @defer:
 * - main.js: 200 KB (critical path)
 * - feature-grid.js: 80 KB (deferred)
 * - testimonials.js: 60 KB (deferred)
 * - pricing-table.js: 40 KB (deferred)
 * - contact-form.js: 50 KB (deferred)
 * - footer.js: 70 KB (deferred)
 * - Total: 500 KB (same)
 * - Initial load: 1s (60% faster!)
 * 
 * Benefits:
 * - 60% smaller initial bundle
 * - 60% faster initial load
 * - Better First Contentful Paint
 * - Improved Time to Interactive
 */
```

## Common Mistakes

### 1. Over-Deferring Critical Content

```typescript
// BAD: Critical content deferred
@Component({
  template: `
    @defer (on timer(1s)) {
      <h1>Welcome!</h1>
      <p>Main content</p>
    }
  `
})
export class BadComponent {}

// GOOD: Critical content immediate
@Component({
  template: `
    <h1>Welcome!</h1>
    <p>Main content</p>
    
    @defer (on viewport) {
      <app-secondary-content></app-secondary-content>
    }
  `
})
export class GoodComponent {}
```

### 2. Not Using Prefetch

```typescript
// BAD: No prefetch, slow interaction
@Component({
  template: `
    <button #btn>Show Modal</button>
    
    @defer (on interaction(btn)) {
      <app-modal></app-modal>
    }
  `
})
export class BadComponent {}

// GOOD: Prefetch for better UX
@Component({
  template: `
    <button #btn>Show Modal</button>
    
    @defer (on interaction(btn); prefetch on hover(btn)) {
      <app-modal></app-modal>
    }
  `
})
export class GoodComponent {}
```

### 3. Missing Loading States

```typescript
// BAD: No loading feedback
@Component({
  template: `
    @defer (on viewport) {
      <app-heavy></app-heavy>
    }
  `
})
export class BadComponent {}

// GOOD: Clear loading feedback
@Component({
  template: `
    @defer (on viewport) {
      <app-heavy></app-heavy>
    }
    
    @loading (minimum 500ms) {
      <div class="skeleton">Loading...</div>
    }
    
    @placeholder {
      <div>Scroll to load</div>
    }
  `
})
export class GoodComponent {}
```

## Best Practices

### 1. Defer Below-the-Fold Content

```typescript
@Component({
  template: `
    <!-- Above the fold - immediate -->
    <header>...</header>
    <hero>...</hero>
    
    <!-- Below the fold - deferred -->
    @defer (on viewport) {
      <app-features></app-features>
    }
  `
})
export class BestPracticeComponent {}
```

### 2. Use Prefetch for Interactive Elements

```typescript
@Component({
  template: `
    <button #btn>Open</button>
    
    @defer (
      on interaction(btn); 
      prefetch on hover(btn)
    ) {
      <app-modal></app-modal>
    }
  `
})
export class PrefetchBestPractice {}
```

### 3. Provide Meaningful Loading States

```typescript
@Component({
  template: `
    @defer (on viewport) {
      <app-content></app-content>
    }
    
    @loading (minimum 300ms) {
      <div class="skeleton">
        <!-- Match content layout -->
        <div class="skeleton-header"></div>
        <div class="skeleton-body"></div>
      </div>
    }
  `
})
export class LoadingBestPractice {}
```

### 4. Handle Errors Gracefully

```typescript
@Component({
  template: `
    @defer (when condition) {
      <app-content></app-content>
    }
    
    @error {
      <div class="error">
        <p>Failed to load</p>
        <button (click)="retry()">Retry</button>
      </div>
    }
  `
})
export class ErrorBestPractice {
  retry() { /* retry logic */ }
}
```

## Interview Questions

### Q1: What are defer blocks in Angular?

**Answer:** Defer blocks (@defer) are a template syntax for declarative lazy loading. They allow you to specify when components should be loaded using triggers like idle, viewport, interaction, timer, or conditions.

### Q2: What's the difference between @defer and lazy loaded routes?

**Answer:** Lazy loaded routes split code at the route level. @defer splits code at the component level within a route, providing more granular control over what loads and when.

### Q3: When should you use prefetch?

**Answer:** Use prefetch when you expect users will need the content soon. For example, prefetch on hover for interactive elements, or prefetch on idle for likely-to-be-viewed content.

### Q4: What are the different defer triggers?

**Answer:** idle, viewport, interaction (click/hover/focus), timer, immediate, and when (conditional). Multiple triggers can be combined with semicolons.

### Q5: How does minimum loading duration work?

**Answer:** minimum ensures loading state shows for at least the specified duration, preventing loading flashes for fast-loading content. Use 300-500ms for a smooth experience.

### Q6: Can you nest defer blocks?

**Answer:** Yes, you can nest defer blocks for progressive loading. Outer blocks load first, then inner blocks load based on their triggers.

### Q7: What happens if a deferred component fails to load?

**Answer:** The @error block is shown. You should provide error recovery options like retry buttons or fallback content.

### Q8: How do defer blocks impact bundle size?

**Answer:** Defer blocks create separate chunks for deferred content, reducing the main bundle size. Initial load is faster, with deferred content loaded on demand.

## Key Takeaways

1. **@defer enables declarative lazy loading** directly in templates
2. **Multiple trigger types available**: idle, viewport, interaction, timer, when
3. **Use prefetch to improve perceived performance** by loading before showing
4. **Always provide loading and error states** for better UX
5. **minimum loading duration prevents loading flashes** (use 300-500ms)
6. **after parameter delays loading indicators** for fast loads
7. **Defer below-the-fold content** to optimize initial load
8. **Can combine multiple triggers** with semicolons
9. **Nest defer blocks** for progressive loading patterns
10. **Measure bundle impact** with build analyzer to verify improvements

## Resources

### Official Documentation

- [Angular Defer Blocks](https://angular.dev/guide/defer)
- [Deferrable Views API](https://angular.dev/api/core/defer)
- [Performance Guide](https://angular.dev/best-practices/runtime-performance)

### Articles

- "Defer Blocks Deep Dive" - Angular Blog
- "Optimizing with @defer" - Web.dev
- "Lazy Loading Strategies" - Angular University

### Video Tutorials

- "Introducing Defer Blocks" - ng-conf
- "Performance with Defer" - Angular Connect
- "Advanced Defer Patterns" - Angular YouTube

### Tools

- Angular CLI - Built-in defer support
- Bundle Analyzer - Measure chunk sizes
- Lighthouse - Measure performance impact
