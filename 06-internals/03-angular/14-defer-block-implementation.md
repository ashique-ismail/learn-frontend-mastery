# Defer Block Implementation

## The Idea

**In plain English:** A defer block is a way to tell Angular "don't load this part of the page yet — wait until it's actually needed." This saves time on the first load because the browser only downloads the code for parts of the page the user is about to see or interact with.

**Real-world analogy:** Imagine a restaurant that only plates and brings out your dessert when you finish your main course — not when you first sit down. The kitchen knows it will be needed eventually, but there's no point carrying it to the table before you're ready.

- The kitchen preparing dessert = Angular downloading and compiling a component's code
- "Only bring it when the main course is finished" = the trigger condition (e.g., `on viewport` or `on interaction`)
- The placeholder napkin folded on the dessert plate = the `@placeholder` block shown before the real content arrives

---

## Table of Contents
- [Introduction](#introduction)
- [Defer Block Basics](#defer-block-basics)
- [Trigger Conditions](#trigger-conditions)
- [Prefetch Strategies](#prefetch-strategies)
- [Loading and Error States](#loading-and-error-states)
- [Implementation Internals](#implementation-internals)
- [Placeholder Management](#placeholder-management)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Defer blocks are Angular's built-in solution for lazy loading components and their dependencies. Introduced in Angular 17, defer blocks provide declarative syntax for deferring the loading of components until they're needed, significantly improving initial load performance.

Key concepts:
- **Declarative lazy loading**: Use `@defer` syntax in templates
- **Smart triggers**: Multiple trigger conditions (viewport, idle, interaction, timer)
- **Prefetching**: Load dependencies before they're displayed
- **State management**: Handle loading, error, and placeholder states
- **Dependency tracking**: Automatically track and lazy load component dependencies

## Defer Block Basics

### Basic Syntax

The fundamental structure of defer blocks:

```typescript
@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="dashboard">
      <h1>Dashboard</h1>
      
      <!-- Basic defer block -->
      @defer {
        <app-heavy-chart [data]="chartData"></app-heavy-chart>
      } @placeholder {
        <div class="chart-skeleton">Loading chart...</div>
      }
      
      <!-- Defer with loading state -->
      @defer {
        <app-user-list [users]="users"></app-user-list>
      } @loading {
        <div class="spinner"></div>
      } @placeholder {
        <p>User list will load soon</p>
      } @error {
        <p>Failed to load users</p>
      }
    </div>
  `
})
export class DashboardComponent {
  chartData = [/* ... */];
  users = [/* ... */];
}
```

### Component Dependencies

Defer blocks automatically track dependencies:

```typescript
// Heavy component that will be lazy loaded
@Component({
  selector: 'app-analytics-widget',
  standalone: true,
  imports: [
    ChartsModule,        // Heavy charting library
    DataTableModule,     // Large table component
    ExportModule         // PDF/Excel export
  ],
  template: `
    <div class="analytics">
      <app-chart [data]="data"></app-chart>
      <app-data-table [rows]="rows"></app-data-table>
      <app-export-buttons></app-export-buttons>
    </div>
  `
})
export class AnalyticsWidgetComponent {
  @Input() data: any;
  @Input() rows: any[];
}

// Parent component with defer
@Component({
  selector: 'app-admin',
  standalone: true,
  template: `
    <div class="admin-panel">
      <h1>Admin Dashboard</h1>
      
      <!-- All dependencies of AnalyticsWidgetComponent -->
      <!-- (ChartsModule, DataTableModule, ExportModule) -->
      <!-- are lazy loaded together -->
      @defer (on viewport) {
        <app-analytics-widget 
          [data]="analyticsData"
          [rows]="tableRows">
        </app-analytics-widget>
      } @placeholder {
        <div class="analytics-placeholder">
          Analytics loading...
        </div>
      }
    </div>
  `
})
export class AdminComponent {
  analyticsData = [/* ... */];
  tableRows = [/* ... */];
}
```

### Nested Defer Blocks

Multiple levels of deferral:

```typescript
@Component({
  selector: 'app-page',
  template: `
    <div class="page">
      <!-- First level: Load main content -->
      @defer (on idle) {
        <app-main-content>
          <!-- Second level: Load comments when visible -->
          @defer (on viewport) {
            <app-comments [postId]="postId"></app-comments>
          } @placeholder {
            <div class="comments-placeholder">
              Comments will appear here
            </div>
          }
        </app-main-content>
      } @placeholder {
        <div class="content-skeleton"></div>
      }
    </div>
  `
})
export class PageComponent {
  postId = 123;
}
```

## Trigger Conditions

### On Idle Trigger

Load when browser is idle:

```typescript
@Component({
  template: `
    <!-- Load when browser has free CPU time -->
    @defer (on idle) {
      <app-recommendations [userId]="userId"></app-recommendations>
    } @placeholder {
      <div class="placeholder">Recommendations loading...</div>
    }
  `
})
export class HomeComponent {
  userId = 'user-123';
}

// Implementation uses requestIdleCallback
class IdleTrigger {
  private idleCallbackId?: number;

  observe(callback: () => void): void {
    if ('requestIdleCallback' in window) {
      this.idleCallbackId = requestIdleCallback(() => {
        callback();
      }, { timeout: 2000 }); // Fallback after 2s
    } else {
      // Fallback for browsers without requestIdleCallback
      setTimeout(callback, 0);
    }
  }

  cleanup(): void {
    if (this.idleCallbackId) {
      cancelIdleCallback(this.idleCallbackId);
    }
  }
}
```

### On Viewport Trigger

Load when element enters viewport:

```typescript
@Component({
  template: `
    <!-- Load when placeholder enters viewport -->
    @defer (on viewport) {
      <app-product-details [productId]="productId"></app-product-details>
    } @placeholder {
      <div class="product-placeholder">Product loading...</div>
    }
    
    <!-- With custom target element -->
    @defer (on viewport(triggerElement)) {
      <app-large-image [src]="imageSrc"></app-large-image>
    } @placeholder {
      <div #triggerElement class="image-placeholder">
        Image loads when this enters viewport
      </div>
    }
  `
})
export class ProductPageComponent {
  productId = 'prod-456';
  imageSrc = 'large-image.jpg';
}

// Implementation uses IntersectionObserver
class ViewportTrigger {
  private observer?: IntersectionObserver;

  observe(element: Element, callback: () => void): void {
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            callback();
            this.cleanup();
          }
        });
      },
      {
        threshold: 0,      // Trigger as soon as any part is visible
        rootMargin: '0px'  // No margin
      }
    );

    this.observer.observe(element);
  }

  cleanup(): void {
    this.observer?.disconnect();
  }
}
```

### On Interaction Trigger

Load on user interaction:

```typescript
@Component({
  template: `
    <!-- Load on click -->
    @defer (on interaction) {
      <app-dialog [content]="dialogContent"></app-dialog>
    } @placeholder {
      <button>Click to open dialog</button>
    }
    
    <!-- Load on hover -->
    @defer (on hover) {
      <app-tooltip [text]="tooltipText"></app-tooltip>
    } @placeholder {
      <span class="tooltip-trigger">Hover me</span>
    }
    
    <!-- Custom trigger element -->
    @defer (on interaction(triggerBtn)) {
      <app-modal [title]="modalTitle"></app-modal>
    } @placeholder {
      <button #triggerBtn>Open Modal</button>
    }
  `
})
export class InteractiveComponent {
  dialogContent = 'Dialog content';
  tooltipText = 'Tooltip text';
  modalTitle = 'Modal Title';
}

// Implementation
class InteractionTrigger {
  private eventListeners: Array<[Element, string, () => void]> = [];

  observe(element: Element, type: 'click' | 'hover', callback: () => void): void {
    const events = type === 'hover' 
      ? ['mouseenter', 'focus']
      : ['click'];

    events.forEach(eventName => {
      const handler = () => {
        callback();
        this.cleanup();
      };

      element.addEventListener(eventName, handler);
      this.eventListeners.push([element, eventName, handler]);
    });
  }

  cleanup(): void {
    this.eventListeners.forEach(([element, event, handler]) => {
      element.removeEventListener(event, handler);
    });
    this.eventListeners = [];
  }
}
```

### On Timer Trigger

Load after a delay:

```typescript
@Component({
  template: `
    <!-- Load after 2 seconds -->
    @defer (on timer(2000)) {
      <app-newsletter-signup></app-newsletter-signup>
    } @placeholder {
      <div class="newsletter-placeholder"></div>
    }
    
    <!-- Load after 5 seconds -->
    @defer (on timer(5s)) {
      <app-survey-prompt></app-survey-prompt>
    } @placeholder {
      <div></div>
    }
  `
})
export class MarketingComponent {}

// Implementation
class TimerTrigger {
  private timerId?: number;

  observe(delayMs: number, callback: () => void): void {
    this.timerId = window.setTimeout(callback, delayMs);
  }

  cleanup(): void {
    if (this.timerId) {
      clearTimeout(this.timerId);
    }
  }
}
```

### On Immediate Trigger

Load immediately (for testing/special cases):

```typescript
@Component({
  template: `
    <!-- Load immediately (no lazy loading) -->
    @defer (on immediate) {
      <app-critical-content></app-critical-content>
    }
    
    <!-- Useful for conditional immediate loading -->
    @defer (on immediate) {
      @if (showContent) {
        <app-content></app-content>
      }
    }
  `
})
export class ImmediateComponent {
  showContent = true;
}
```

### Multiple Triggers

Combine multiple trigger conditions:

```typescript
@Component({
  template: `
    <!-- Load on ANY of these conditions -->
    @defer (on idle; on timer(3000); on viewport) {
      <app-analytics-tracker></app-analytics-tracker>
    } @placeholder {
      <div></div>
    }
    
    <!-- Load on viewport OR interaction -->
    @defer (on viewport; on interaction) {
      <app-video-player [src]="videoSrc"></app-video-player>
    } @placeholder {
      <div class="video-placeholder">
        <button>Play Video</button>
      </div>
    }
  `
})
export class MultiTriggerComponent {
  videoSrc = 'video.mp4';
}

// Implementation combines triggers
class CompositeTrigger {
  private triggers: Array<BaseTrigger> = [];
  private triggered = false;

  addTrigger(trigger: BaseTrigger): void {
    this.triggers.push(trigger);
  }

  observe(callback: () => void): void {
    // Set up all triggers
    this.triggers.forEach(trigger => {
      trigger.observe(() => {
        if (!this.triggered) {
          this.triggered = true;
          callback();
          this.cleanup();
        }
      });
    });
  }

  cleanup(): void {
    this.triggers.forEach(trigger => trigger.cleanup());
    this.triggers = [];
  }
}
```

## Prefetch Strategies

### When to Prefetch

Prefetch dependencies before they're displayed:

```typescript
@Component({
  template: `
    <!-- Prefetch on idle, load on viewport -->
    @defer (on viewport; prefetch on idle) {
      <app-product-grid [products]="products"></app-product-grid>
    } @placeholder {
      <div class="grid-skeleton"></div>
    }
    
    <!-- Prefetch on hover, load on click -->
    @defer (on interaction; prefetch on hover) {
      <app-detailed-modal [data]="modalData"></app-detailed-modal>
    } @placeholder {
      <button>View Details</button>
    }
    
    <!-- Prefetch after timer, load on viewport -->
    @defer (on viewport; prefetch on timer(1000)) {
      <app-comments [postId]="postId"></app-comments>
    } @placeholder {
      <div class="comments-placeholder">Comments</div>
    }
  `
})
export class PrefetchExampleComponent {
  products = [/* ... */];
  modalData = { /* ... */ };
  postId = 123;
}
```

### Prefetch Implementation

How prefetch works internally:

```typescript
// Prefetch manager
class DeferBlockPrefetchManager {
  private prefetchedBlocks = new Set<string>();
  private prefetchPromises = new Map<string, Promise<any>>();

  async prefetchBlock(blockId: string, loader: () => Promise<any>): Promise<void> {
    // Check if already prefetched
    if (this.prefetchedBlocks.has(blockId)) {
      return;
    }

    // Check if prefetch in progress
    if (this.prefetchPromises.has(blockId)) {
      return this.prefetchPromises.get(blockId);
    }

    // Start prefetch
    const prefetchPromise = loader().then(() => {
      this.prefetchedBlocks.add(blockId);
      this.prefetchPromises.delete(blockId);
    });

    this.prefetchPromises.set(blockId, prefetchPromise);
    return prefetchPromise;
  }

  async loadBlock(
    blockId: string,
    loader: () => Promise<any>
  ): Promise<any> {
    // If already prefetched, load immediately
    if (this.prefetchedBlocks.has(blockId)) {
      return Promise.resolve();
    }

    // If prefetch in progress, wait for it
    if (this.prefetchPromises.has(blockId)) {
      await this.prefetchPromises.get(blockId);
      return Promise.resolve();
    }

    // Otherwise, load now
    return loader();
  }
}
```

### Strategic Prefetching

Smart prefetch patterns:

```typescript
@Component({
  selector: 'app-article',
  template: `
    <article>
      <h1>{{ article.title }}</h1>
      <div [innerHTML]="article.content"></div>
      
      <!-- Prefetch related articles on idle -->
      <!-- User might scroll to them -->
      @defer (on viewport; prefetch on idle) {
        <app-related-articles [articleId]="article.id">
        </app-related-articles>
      } @placeholder {
        <div class="related-placeholder">Related Articles</div>
      }
      
      <!-- Prefetch comments on timer -->
      <!-- Users often read article before scrolling to comments -->
      @defer (on viewport; prefetch on timer(2000)) {
        <app-comment-section [articleId]="article.id">
        </app-comment-section>
      } @placeholder {
        <div class="comments-placeholder">Comments</div>
      }
      
      <!-- Prefetch share dialog on hover -->
      <!-- User hovering suggests they might share -->
      @defer (on interaction(shareBtn); prefetch on hover(shareBtn)) {
        <app-share-dialog [url]="article.url"></app-share-dialog>
      } @placeholder {
        <button #shareBtn>Share</button>
      }
    </article>
  `
})
export class ArticleComponent {
  @Input() article!: Article;
}
```

## Loading and Error States

### Loading State Management

Handle loading transitions:

```typescript
@Component({
  template: `
    <!-- Minimum loading duration -->
    @defer (on viewport) {
      <app-content></app-content>
    } @loading (minimum 500ms) {
      <div class="spinner">Loading...</div>
    } @placeholder {
      <div class="placeholder"></div>
    }
    
    <!-- Loading after delay -->
    <!-- Don't show loading for fast loads -->
    @defer (on interaction) {
      <app-modal></app-modal>
    } @loading (after 200ms; minimum 500ms) {
      <div class="modal-loading">Please wait...</div>
    } @placeholder {
      <button>Open</button>
    }
  `
})
export class LoadingStatesComponent {}

// Implementation
class LoadingStateManager {
  private loadingStartTime?: number;
  private minimumLoadingTime = 0;
  private afterDelay = 0;
  private showLoadingTimer?: number;

  configure(options: { minimum?: number; after?: number }): void {
    this.minimumLoadingTime = options.minimum || 0;
    this.afterDelay = options.after || 0;
  }

  async startLoading(
    showLoading: () => void,
    hideLoading: () => void
  ): Promise<void> {
    this.loadingStartTime = Date.now();

    // Show loading after delay
    if (this.afterDelay > 0) {
      this.showLoadingTimer = window.setTimeout(() => {
        showLoading();
      }, this.afterDelay);
    } else {
      showLoading();
    }

    return new Promise(resolve => {
      // Ensure minimum loading time
      const elapsed = Date.now() - this.loadingStartTime!;
      const remaining = Math.max(0, this.minimumLoadingTime - elapsed);

      setTimeout(() => {
        if (this.showLoadingTimer) {
          clearTimeout(this.showLoadingTimer);
        }
        hideLoading();
        resolve();
      }, remaining);
    });
  }
}
```

### Error Handling

Manage load failures:

```typescript
@Component({
  template: `
    <!-- Basic error handling -->
    @defer (on viewport) {
      <app-widget [data]="widgetData"></app-widget>
    } @error {
      <div class="error-state">
        <p>Failed to load widget</p>
        <button (click)="retryLoad()">Retry</button>
      </div>
    } @placeholder {
      <div class="widget-placeholder"></div>
    }
    
    <!-- Custom error with details -->
    @defer (on interaction) {
      <app-report-generator></app-report-generator>
    } @error {
      <app-error-display 
        [error]="lastError"
        (retry)="retryLoad()">
      </app-error-display>
    } @placeholder {
      <button>Generate Report</button>
    }
  `
})
export class ErrorHandlingComponent {
  widgetData = [/* ... */];
  lastError?: Error;

  retryLoad(): void {
    // Trigger reload logic
    console.log('Retrying load...');
  }
}

// Error state manager
class DeferBlockErrorManager {
  private errorHandlers = new Map<string, (error: Error) => void>();

  handleLoadError(blockId: string, error: Error): void {
    console.error(`Defer block ${blockId} failed to load:`, error);

    const handler = this.errorHandlers.get(blockId);
    if (handler) {
      handler(error);
    }

    // Report to error tracking service
    this.reportError(blockId, error);
  }

  registerErrorHandler(
    blockId: string,
    handler: (error: Error) => void
  ): void {
    this.errorHandlers.set(blockId, handler);
  }

  private reportError(blockId: string, error: Error): void {
    // Send to error tracking (Sentry, etc.)
  }
}
```

### State Transitions

Complete state flow:

```typescript
@Component({
  template: `
    <!-- All states defined -->
    @defer (on viewport) {
      <app-complete-widget [data]="data"></app-complete-widget>
    } @placeholder {
      <!-- Initial state: Before any trigger -->
      <div class="placeholder">
        <p>Widget will load when visible</p>
      </div>
    } @loading (after 100ms; minimum 300ms) {
      <!-- Loading state: Dependencies are loading -->
      <div class="loading-state">
        <div class="spinner"></div>
        <p>Loading widget...</p>
      </div>
    } @error {
      <!-- Error state: Load failed -->
      <div class="error-state">
        <p>Failed to load widget</p>
        <button (click)="retry()">Try Again</button>
      </div>
    }
  `
})
export class CompleteStatesComponent {
  data = [/* ... */];

  retry(): void {
    // Implement retry logic
  }
}

// State machine
enum DeferBlockState {
  Placeholder,
  Loading,
  Loaded,
  Error
}

class DeferBlockStateMachine {
  private state = DeferBlockState.Placeholder;

  transitionTo(newState: DeferBlockState): void {
    console.log(`State transition: ${this.state} -> ${newState}`);
    this.state = newState;
  }

  canTransition(to: DeferBlockState): boolean {
    // Define valid transitions
    const validTransitions = {
      [DeferBlockState.Placeholder]: [DeferBlockState.Loading],
      [DeferBlockState.Loading]: [DeferBlockState.Loaded, DeferBlockState.Error],
      [DeferBlockState.Loaded]: [],
      [DeferBlockState.Error]: [DeferBlockState.Loading] // Allow retry
    };

    return validTransitions[this.state].includes(to);
  }

  getCurrentState(): DeferBlockState {
    return this.state;
  }
}
```

## Implementation Internals

### Compilation Output

How defer blocks compile:

```typescript
// Source code
@Component({
  template: `
    @defer (on viewport) {
      <app-lazy [data]="data"></app-lazy>
    } @placeholder {
      <div>Placeholder</div>
    }
  `
})
export class MyComponent {
  data = [];
}

// Compiled output (simplified)
class MyComponent {
  static ɵcmp = defineComponent({
    // ...
    template: function MyComponent_Template(rf, ctx) {
      if (rf & 1) {
        // Create defer block
        ɵɵdefer(
          0,                    // Block ID
          0,                    // Primary template index
          MyComponent_Defer_0,  // Primary template function
          1,                    // Placeholder template index
          MyComponent_Defer_0_Placeholder, // Placeholder function
          null,                 // Loading template (none)
          null,                 // Error template (none)
          [                     // Triggers
            ɵɵdeferOnViewport()
          ]
        );
      }
    },
    dependencies: () => [
      // Lazy component imported dynamically
      () => import('./lazy.component').then(m => m.AppLazyComponent)
    ]
  });
}

// Template functions
function MyComponent_Defer_0(rf, ctx) {
  if (rf & 1) {
    ɵɵelement(0, 'app-lazy', ['data', ctx.data]);
  }
}

function MyComponent_Defer_0_Placeholder(rf, ctx) {
  if (rf & 1) {
    ɵɵelementStart(0, 'div');
    ɵɵtext(1, 'Placeholder');
    ɵɵelementEnd();
  }
}
```

### Defer Block Registry

Managing active defer blocks:

```typescript
// Global defer block registry
class DeferBlockRegistry {
  private blocks = new Map<string, DeferBlock>();

  register(block: DeferBlock): void {
    this.blocks.set(block.id, block);
  }

  unregister(blockId: string): void {
    const block = this.blocks.get(blockId);
    if (block) {
      block.cleanup();
      this.blocks.delete(blockId);
    }
  }

  getBlock(blockId: string): DeferBlock | undefined {
    return this.blocks.get(blockId);
  }

  getAllBlocks(): DeferBlock[] {
    return Array.from(this.blocks.values());
  }
}

// Defer block instance
class DeferBlock {
  id: string;
  state: DeferBlockState;
  triggers: DeferTrigger[];
  prefetchTriggers: DeferTrigger[];
  loader: () => Promise<any>;
  
  private cleanupFns: Array<() => void> = [];

  constructor(config: DeferBlockConfig) {
    this.id = config.id;
    this.state = DeferBlockState.Placeholder;
    this.triggers = config.triggers;
    this.prefetchTriggers = config.prefetchTriggers || [];
    this.loader = config.loader;

    this.setupTriggers();
  }

  private setupTriggers(): void {
    // Setup prefetch triggers
    this.prefetchTriggers.forEach(trigger => {
      trigger.observe(() => this.prefetch());
      this.cleanupFns.push(() => trigger.cleanup());
    });

    // Setup load triggers
    this.triggers.forEach(trigger => {
      trigger.observe(() => this.load());
      this.cleanupFns.push(() => trigger.cleanup());
    });
  }

  async prefetch(): Promise<void> {
    if (this.state !== DeferBlockState.Placeholder) {
      return;
    }

    console.log(`Prefetching defer block: ${this.id}`);
    await this.loader();
  }

  async load(): Promise<void> {
    if (this.state !== DeferBlockState.Placeholder) {
      return;
    }

    this.state = DeferBlockState.Loading;

    try {
      await this.loader();
      this.state = DeferBlockState.Loaded;
    } catch (error) {
      this.state = DeferBlockState.Error;
      console.error(`Failed to load defer block ${this.id}:`, error);
    }
  }

  cleanup(): void {
    this.cleanupFns.forEach(fn => fn());
    this.cleanupFns = [];
  }
}
```

### Dynamic Loading

How components are loaded dynamically:

```typescript
// Lazy component loader
class LazyComponentLoader {
  private cache = new Map<string, Promise<Type<any>>>();

  async loadComponent(
    importFn: () => Promise<any>,
    componentName: string
  ): Promise<Type<any>> {
    const cacheKey = `${importFn.toString()}-${componentName}`;

    // Return cached if available
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }

    // Load component
    const loadPromise = importFn().then(module => {
      const component = module[componentName];
      
      if (!component) {
        throw new Error(`Component ${componentName} not found in module`);
      }

      return component;
    });

    this.cache.set(cacheKey, loadPromise);
    return loadPromise;
  }

  clearCache(): void {
    this.cache.clear();
  }
}

// Usage in defer block
async function loadDeferredDependencies(
  componentImport: () => Promise<any>
): Promise<Type<any>[]> {
  const loader = new LazyComponentLoader();

  // Load main component
  const module = await componentImport();
  const componentType = Object.values(module)[0] as Type<any>;

  // Load component dependencies
  const dependencies: Type<any>[] = [componentType];

  // Recursively load nested dependencies
  const componentDef = getComponentDef(componentType);
  if (componentDef?.dependencies) {
    const nestedDeps = await Promise.all(
      componentDef.dependencies
        .filter(dep => typeof dep === 'function')
        .map(dep => (dep as any)())
    );

    dependencies.push(...nestedDeps.flat());
  }

  return dependencies;
}
```

## Placeholder Management

### Placeholder Rendering

How placeholders work:

```typescript
@Component({
  template: `
    <!-- Simple placeholder -->
    @defer (on viewport) {
      <app-content></app-content>
    } @placeholder {
      <div class="skeleton"></div>
    }
    
    <!-- Minimum display time -->
    <!-- Prevents flicker on fast loads -->
    @defer (on interaction) {
      <app-modal></app-modal>
    } @placeholder (minimum 500ms) {
      <button>Click Me</button>
    }
  `
})
export class PlaceholderComponent {}

// Placeholder manager
class PlaceholderManager {
  private minimumDisplayTime = 0;
  private displayStartTime?: number;

  setMinimumDisplayTime(ms: number): void {
    this.minimumDisplayTime = ms;
  }

  startDisplaying(): void {
    this.displayStartTime = Date.now();
  }

  async canHide(): Promise<void> {
    if (!this.displayStartTime || this.minimumDisplayTime === 0) {
      return Promise.resolve();
    }

    const elapsed = Date.now() - this.displayStartTime;
    const remaining = Math.max(0, this.minimumDisplayTime - elapsed);

    if (remaining > 0) {
      return new Promise(resolve => {
        setTimeout(resolve, remaining);
      });
    }

    return Promise.resolve();
  }
}
```

### Skeleton Screens

Creating effective placeholders:

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    @defer (on viewport) {
      <div class="user-card">
        <img [src]="user.avatar" [alt]="user.name">
        <h3>{{ user.name }}</h3>
        <p>{{ user.bio }}</p>
        <button>Follow</button>
      </div>
    } @placeholder {
      <div class="user-card-skeleton">
        <div class="skeleton-avatar"></div>
        <div class="skeleton-text skeleton-title"></div>
        <div class="skeleton-text skeleton-bio"></div>
        <div class="skeleton-button"></div>
      </div>
    }
  `,
  styles: [`
    .user-card, .user-card-skeleton {
      padding: 16px;
      border: 1px solid #ddd;
      border-radius: 8px;
    }

    .skeleton-avatar {
      width: 64px;
      height: 64px;
      border-radius: 50%;
      background: linear-gradient(
        90deg,
        #f0f0f0 25%,
        #e0e0e0 50%,
        #f0f0f0 75%
      );
      background-size: 200% 100%;
      animation: shimmer 1.5s infinite;
    }

    .skeleton-text {
      height: 16px;
      margin: 8px 0;
      border-radius: 4px;
      background: linear-gradient(
        90deg,
        #f0f0f0 25%,
        #e0e0e0 50%,
        #f0f0f0 75%
      );
      background-size: 200% 100%;
      animation: shimmer 1.5s infinite;
    }

    .skeleton-title {
      width: 60%;
    }

    .skeleton-bio {
      width: 80%;
    }

    .skeleton-button {
      width: 80px;
      height: 32px;
      margin-top: 12px;
      border-radius: 4px;
      background: linear-gradient(
        90deg,
        #f0f0f0 25%,
        #e0e0e0 50%,
        #f0f0f0 75%
      );
      background-size: 200% 100%;
      animation: shimmer 1.5s infinite;
    }

    @keyframes shimmer {
      0% {
        background-position: 200% 0;
      }
      100% {
        background-position: -200% 0;
      }
    }
  `]
})
export class UserCardComponent {
  @Input() user!: User;
}
```

## Common Misconceptions

### Misconception 1: "Defer blocks automatically split code"

**Reality**: Code splitting happens at the component/module level. Defer blocks trigger the loading, but you need proper module organization.

```typescript
// NOT automatically code-split
@Component({
  standalone: true,
  imports: [HeavyComponent], // Imported statically
  template: `
    @defer (on viewport) {
      <app-heavy></app-heavy>
    }
  `
})
export class WrongApproach {}

// CORRECT - Component is standalone and loaded dynamically
@Component({
  standalone: true,
  template: `
    @defer (on viewport) {
      <!-- HeavyComponent is loaded dynamically -->
      <app-heavy></app-heavy>
    }
  `
})
export class CorrectApproach {}
```

### Misconception 2: "Prefetch always improves performance"

**Reality**: Prefetch uses bandwidth and memory. Only prefetch when likely to be used.

```typescript
// BAD - Prefetching everything wastes resources
@Component({
  template: `
    @defer (on viewport; prefetch on immediate) {
      <app-rarely-used></app-rarely-used>
    }
  `
})
export class BadPrefetch {}

// GOOD - Strategic prefetching
@Component({
  template: `
    <!-- Prefetch on idle - user likely to scroll -->
    @defer (on viewport; prefetch on idle) {
      <app-below-fold></app-below-fold>
    }
    
    <!-- Don't prefetch - rarely accessed -->
    @defer (on interaction) {
      <app-admin-panel></app-admin-panel>
    }
  `
})
export class GoodPrefetch {}
```

### Misconception 3: "Loading state is always needed"

**Reality**: For fast loads, showing loading state causes flicker.

```typescript
// Shows loading even for instant loads - causes flicker
@defer (on viewport) {
  <app-content></app-content>
} @loading {
  <div class="spinner"></div>
}

// Better - only show loading if it takes more than 200ms
@defer (on viewport) {
  <app-content></app-content>
} @loading (after 200ms; minimum 300ms) {
  <div class="spinner"></div>
}
```

## Performance Implications

### Bundle Size Optimization

Measuring defer block impact:

```typescript
// Before defer blocks:
// main.bundle.js: 1.2 MB
// - App code: 200 KB
// - Angular: 300 KB
// - Heavy components: 700 KB

// After defer blocks:
// main.bundle.js: 500 KB (58% reduction!)
// - App code: 200 KB
// - Angular: 300 KB
// lazy-component-1.js: 200 KB (loaded on demand)
// lazy-component-2.js: 250 KB (loaded on demand)
// lazy-component-3.js: 250 KB (loaded on demand)

// Measuring impact
@Injectable()
export class PerformanceMonitor {
  private initialBundleSize = 0;
  private lazyBundleSize = 0;

  logBundleLoad(name: string, size: number): void {
    if (name === 'main') {
      this.initialBundleSize = size;
    } else {
      this.lazyBundleSize += size;
    }

    console.log(`Bundle ${name}: ${size} KB`);
    console.log(`Total lazy: ${this.lazyBundleSize} KB`);
    console.log(`Savings: ${this.lazyBundleSize} KB not in initial bundle`);
  }
}
```

### Loading Performance

Optimizing load times:

```typescript
@Component({
  template: `
    <!-- Strategy 1: Viewport-based lazy loading -->
    <!-- Only load when visible -->
    @defer (on viewport) {
      <app-below-fold-content></app-below-fold-content>
    }
    
    <!-- Strategy 2: Idle-based loading -->
    <!-- Load during browser idle time -->
    @defer (on idle) {
      <app-low-priority-widget></app-low-priority-widget>
    }
    
    <!-- Strategy 3: Interaction-based loading -->
    <!-- Only load when user shows intent -->
    @defer (on hover; prefetch on hover) {
      <app-heavy-dropdown></app-heavy-dropdown>
    }
    
    <!-- Strategy 4: Timer-based loading -->
    <!-- Load after critical content is ready -->
    @defer (on timer(2000)) {
      <app-analytics-tracker></app-analytics-tracker>
    }
  `
})
export class OptimizedComponent {}
```

### Memory Management

Cleanup and resource management:

```typescript
@Injectable()
export class DeferBlockMemoryManager {
  private loadedBlocks = new Set<string>();
  private memoryUsage = new Map<string, number>();

  trackBlockLoad(blockId: string, estimatedSize: number): void {
    this.loadedBlocks.add(blockId);
    this.memoryUsage.set(blockId, estimatedSize);

    console.log(`Total defer blocks loaded: ${this.loadedBlocks.size}`);
    console.log(`Estimated memory: ${this.getTotalMemory()} KB`);
  }

  getTotalMemory(): number {
    return Array.from(this.memoryUsage.values()).reduce((a, b) => a + b, 0);
  }

  shouldUnloadBlock(blockId: string): boolean {
    // Unload if total memory exceeds threshold
    const threshold = 10000; // 10 MB
    return this.getTotalMemory() > threshold;
  }
}
```

## Interview Questions

### Question 1: What are defer blocks and how do they improve performance?

**Answer**: Defer blocks are Angular's declarative lazy loading syntax introduced in Angular 17. They improve performance by:

1. **Reducing initial bundle size**: Deferred components aren't in the main bundle
2. **Faster initial load**: Less JavaScript to download and parse
3. **On-demand loading**: Components load only when needed
4. **Better resource utilization**: Load during idle time or when visible

```typescript
@defer (on viewport) {
  <app-heavy-component></app-heavy-component>
} @placeholder {
  <div>Loading...</div>
}
```

### Question 2: What are the different trigger conditions for defer blocks?

**Answer**: Angular provides several triggers:

1. **on idle**: Load when browser is idle (requestIdleCallback)
2. **on viewport**: Load when element enters viewport (IntersectionObserver)
3. **on interaction**: Load on click or hover
4. **on hover**: Load on mouseenter
5. **on timer(ms)**: Load after delay
6. **on immediate**: Load immediately (no lazy loading)

Multiple triggers can be combined: `@defer (on viewport; on timer(3000))`

### Question 3: How does prefetching work in defer blocks?

**Answer**: Prefetching loads dependencies before they're displayed:

```typescript
@defer (on viewport; prefetch on idle) {
  <app-component></app-component>
}
```

This prefetches on idle (downloads and compiles) but only displays when in viewport. Benefits:
- Faster display when trigger occurs
- Still reduces initial bundle size
- Uses idle browser time efficiently

Best for components likely to be used soon.

### Question 4: What's the purpose of loading and error blocks?

**Answer**: They handle loading states gracefully:

```typescript
@defer (on viewport) {
  <app-content></app-content>
} @loading (after 200ms; minimum 500ms) {
  <div>Loading...</div>
} @error {
  <div>Failed to load</div>
}
```

- **after**: Delay showing loading (prevents flicker)
- **minimum**: Keep loading visible minimum time (prevents flicker)
- **error**: Handle load failures gracefully

### Question 5: Can defer blocks be nested? What are the implications?

**Answer**: Yes, defer blocks can be nested for multi-level lazy loading:

```typescript
@defer (on idle) {
  <app-section>
    @defer (on viewport) {
      <app-subsection></app-subsection>
    }
  </app-section>
}
```

Implications:
- Further reduces initial bundle
- More granular loading control
- More complex state management
- Potential for loading waterfalls (mitigate with prefetch)

### Question 6: How do defer blocks affect SEO and server-side rendering?

**Answer**: Defer blocks work with SSR:

1. **Server rendering**: All deferred content is rendered on server
2. **Client hydration**: Angular hydrates based on triggers
3. **SEO-friendly**: Search engines see full content

However:
- Content below fold can be deferred for client-only
- Critical content shouldn't be deferred
- Use `on immediate` for SEO-critical deferred sections

### Question 7: What's the difference between defer blocks and lazy-loaded routes?

**Answer**:

**Lazy-loaded routes**:
- Module/component-level
- Route-based navigation
- Separate chunks per route
- Loaded on route activation

**Defer blocks**:
- Template-level
- Conditional rendering
- Can be within a route
- Multiple triggers (viewport, idle, etc.)

They complement each other:
```typescript
// Lazy route
{ path: 'admin', loadChildren: () => import('./admin/admin.module') }

// Within route, defer blocks for sections
@Component({
  template: `
    @defer (on viewport) {
      <app-admin-analytics></app-admin-analytics>
    }
  `
})
```

### Question 8: How do you debug issues with defer blocks?

**Answer**: Debugging strategies:

1. **Check trigger conditions**: Verify triggers are firing
2. **Use immediate trigger**: Test without lazy loading
3. **Check console**: Look for load errors
4. **Network tab**: Verify chunks are loading
5. **Angular DevTools**: Inspect component tree

```typescript
// Temporary debugging
@defer (on immediate) { // Change from 'on viewport'
  <app-component></app-component>
} @error {
  <div>Error: Check console</div>
}

// Add logging
@defer (on viewport) {
  <app-component></app-component>
} @loading {
  <div>{{ logLoading() }}</div>
}

logLoading() {
  console.log('Component loading...');
  return 'Loading...';
}
```

## Key Takeaways

1. **Defer blocks provide declarative lazy loading** at the template level without routes
2. **Multiple trigger types** enable smart loading strategies (viewport, idle, interaction, timer)
3. **Prefetching loads dependencies early** without immediate rendering, improving perceived performance
4. **Loading states prevent flicker** with after/minimum timing controls
5. **Error handling provides fallbacks** for load failures
6. **Bundle size reduces significantly** by deferring heavy components
7. **Works with SSR** while maintaining SEO benefits
8. **Strategic deferral is key** - don't defer everything, focus on heavy/below-fold content

## Resources

### Official Documentation
- [Angular Defer Blocks Guide](https://angular.dev/guide/defer)
- [Defer Block API](https://angular.dev/api/core/defer)
- [Performance Best Practices](https://angular.dev/best-practices/performance)

### Articles & Tutorials
- "Introducing Defer Blocks in Angular 17" by Angular Team
- "Optimizing Angular Apps with Defer Blocks" by Minko Gechev
- "Lazy Loading Strategies with Defer Blocks" by Netanel Basal

### Videos
- Angular 17: Defer Blocks Deep Dive
- Performance Optimization with Defer Blocks
- Building Fast Angular Apps with Lazy Loading

### Books
- "Angular Performance Optimization Guide"
- "Modern Angular Development Patterns"

### Tools
- Angular DevTools - Inspect defer blocks
- Lighthouse - Measure performance impact
- Bundle Analyzer - Visualize code splitting

### Source Code
- [Defer Block Implementation](https://github.com/angular/angular/tree/main/packages/core/src/defer)
- [Defer Triggers Source](https://github.com/angular/angular/blob/main/packages/core/src/render3/instructions/defer.ts)
