# Angular Directives and Pipes - Interview Questions

## Overview

This guide covers common interview questions about Angular directives and pipes, including structural vs attribute directives, custom directive implementation, pure vs impure pipes, and performance considerations.

## Core Concept Questions

### 1. What's the difference between structural and attribute directives? When would you use each?

**Expected Answer:**

Structural directives change the DOM structure by adding or removing elements:
- Prefixed with asterisk (*ngIf, *ngFor, *ngSwitch)
- Use TemplateRef and ViewContainerRef
- Modify the DOM layout
- Examples: *ngIf, *ngFor, *ngSwitch

Attribute directives change the appearance or behavior of existing elements:
- Applied as attributes without asterisk
- Modify element properties, styles, or classes
- Don't change DOM structure
- Examples: ngClass, ngStyle, ngModel

**Code Example:**

```typescript
// Structural Directive - changes DOM structure
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true
})
export class UnlessDirective {
  private hasView = false;

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      // Create view if condition is false
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      // Remove view if condition is true
      this.viewContainer.clear();
      this.hasView = false;
    }
  }

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}
}

// Usage: <div *appUnless="isLoggedIn">Please log in</div>

// Attribute Directive - changes appearance/behavior
@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';
  @Input() defaultColor = 'transparent';

  constructor(private el: ElementRef, private renderer: Renderer2) {}

  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.appHighlight);
  }

  @HostListener('mouseleave') onMouseLeave() {
    this.highlight(this.defaultColor);
  }

  private highlight(color: string) {
    this.renderer.setStyle(this.el.nativeElement, 'backgroundColor', color);
  }
}

// Usage: <p appHighlight="lightblue">Hover over me</p>
```

**Use Cases:**

Structural directives when you need to:
- Conditionally add/remove elements (*ngIf)
- Repeat elements (*ngFor)
- Switch between multiple templates (*ngSwitch)
- Create custom layout logic

Attribute directives when you need to:
- Change element styling dynamically
- Add/remove CSS classes
- Modify element behavior
- Handle DOM events
- Implement custom validators

**Follow-up Questions:**
1. How does Angular transform the asterisk syntax into ng-template?
2. Can you create a structural directive that provides context variables?
3. How do you handle memory leaks in custom directives?

**What Interviewers Look For:**
- Understanding of when DOM structure changes vs style/behavior changes
- Knowledge of TemplateRef and ViewContainerRef
- Practical examples of when to use each type
- Understanding of the asterisk (*) desugaring process

### 2. How do you create a custom structural directive? Explain TemplateRef and ViewContainerRef.

**Expected Answer:**

TemplateRef represents an embedded template that can be used to instantiate views. ViewContainerRef represents a container where views can be attached.

**Comprehensive Example:**

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  OnInit,
  OnDestroy 
} from '@angular/core';

// Advanced structural directive with context
@Directive({
  selector: '[appRepeat]',
  standalone: true
})
export class RepeatDirective implements OnInit, OnDestroy {
  private viewRefs: any[] = [];

  @Input() appRepeat: number = 0;
  @Input() appRepeatDelay: number = 0;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  ngOnInit() {
    this.render();
  }

  ngOnDestroy() {
    // Clean up all created views
    this.viewRefs.forEach(ref => ref.destroy());
  }

  private render() {
    // Clear existing views
    this.viewContainer.clear();
    this.viewRefs = [];

    // Create views with context
    for (let i = 0; i < this.appRepeat; i++) {
      if (this.appRepeatDelay > 0) {
        setTimeout(() => {
          this.createView(i);
        }, this.appRepeatDelay * i);
      } else {
        this.createView(i);
      }
    }
  }

  private createView(index: number) {
    const context = {
      $implicit: index,
      index: index,
      count: this.appRepeat,
      first: index === 0,
      last: index === this.appRepeat - 1,
      even: index % 2 === 0,
      odd: index % 2 !== 0
    };

    const viewRef = this.viewContainer.createEmbeddedView(
      this.templateRef,
      context
    );
    this.viewRefs.push(viewRef);
  }
}

// Usage with context variables
// <div *appRepeat="5; let i; let isFirst = first; let isEven = even">
//   Item {{i}} - First: {{isFirst}}, Even: {{isEven}}
// </div>

// Permission-based structural directive
@Directive({
  selector: '[appHasPermission]',
  standalone: true
})
export class HasPermissionDirective implements OnInit {
  private permissions: string[] = [];
  private hasView = false;

  @Input() set appHasPermission(permissions: string | string[]) {
    this.permissions = Array.isArray(permissions) ? permissions : [permissions];
    this.updateView();
  }

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  ngOnInit() {
    this.updateView();
  }

  private updateView() {
    const hasPermission = this.permissions.some(permission =>
      this.authService.hasPermission(permission)
    );

    if (hasPermission && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (!hasPermission && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Loading state directive
@Directive({
  selector: '[appLoading]',
  standalone: true
})
export class LoadingDirective {
  private loadingRef: ComponentRef<LoadingSpinnerComponent> | null = null;

  @Input() set appLoading(isLoading: boolean) {
    if (isLoading) {
      this.showLoading();
    } else {
      this.hideLoading();
    }
  }

  constructor(
    private viewContainer: ViewContainerRef,
    private componentFactoryResolver: ComponentFactoryResolver
  ) {}

  private showLoading() {
    if (!this.loadingRef) {
      const factory = this.componentFactoryResolver
        .resolveComponentFactory(LoadingSpinnerComponent);
      this.loadingRef = this.viewContainer.createComponent(factory);
    }
  }

  private hideLoading() {
    if (this.loadingRef) {
      this.loadingRef.destroy();
      this.loadingRef = null;
    }
  }
}
```

**Key Concepts:**

TemplateRef:
- Represents the template content
- Read-only reference
- Used to create embedded views
- Contains the template's structure

ViewContainerRef:
- Container that can hold views
- Provides methods to create/destroy views
- Manages view lifecycle
- Can insert views at specific positions

**Follow-up Questions:**
1. How do you pass multiple input properties to a structural directive?
2. What's the difference between createEmbeddedView and createComponent?
3. How do you handle change detection in custom structural directives?

**What Interviewers Look For:**
- Deep understanding of TemplateRef and ViewContainerRef
- Proper memory management and cleanup
- Knowledge of context variables and how to expose them
- Understanding of view lifecycle

### 3. What's the difference between pure and impure pipes? How do they affect performance?

**Expected Answer:**

Pure pipes execute only when primitive input changes or object reference changes. Impure pipes execute on every change detection cycle.

**Detailed Comparison:**

```typescript
// Pure Pipe (default) - executes only when input changes
@Pipe({
  name: 'filterPure',
  standalone: true,
  pure: true // default
})
export class FilterPurePipe implements PipeTransform {
  transform(items: any[], filterField: string, filterValue: any): any[] {
    console.log('Pure pipe executed'); // Called only when items reference changes
    
    if (!items || !filterField || filterValue === undefined) {
      return items;
    }
    
    return items.filter(item => item[filterField] === filterValue);
  }
}

// Impure Pipe - executes on every change detection
@Pipe({
  name: 'filterImpure',
  standalone: true,
  pure: false
})
export class FilterImpurePipe implements PipeTransform {
  transform(items: any[], filterField: string, filterValue: any): any[] {
    console.log('Impure pipe executed'); // Called on EVERY change detection
    
    if (!items || !filterField || filterValue === undefined) {
      return items;
    }
    
    return items.filter(item => item[filterField] === filterValue);
  }
}

// Component demonstrating the difference
@Component({
  selector: 'app-pipe-demo',
  standalone: true,
  imports: [FilterPurePipe, FilterImpurePipe],
  template: `
    <h3>Pure Pipe</h3>
    <div *ngFor="let item of items | filterPure:'active':true">
      {{item.name}}
    </div>

    <h3>Impure Pipe</h3>
    <div *ngFor="let item of items | filterImpure:'active':true">
      {{item.name}}
    </div>

    <button (click)="addItem()">Add Item</button>
    <button (click)="toggleActive()">Toggle Active</button>
    <button (click)="trackMouse()">{{mousePosition}}</button>
  `
})
export class PipeDemoComponent {
  items = [
    { name: 'Item 1', active: true },
    { name: 'Item 2', active: false }
  ];
  mousePosition = 0;

  addItem() {
    // This won't trigger pure pipe because reference doesn't change
    this.items.push({ name: 'Item 3', active: true });
    
    // To trigger pure pipe, create new reference
    // this.items = [...this.items, { name: 'Item 3', active: true }];
  }

  toggleActive() {
    // This changes property but not array reference
    this.items[0].active = !this.items[0].active;
    
    // Pure pipe won't detect this change
    // Impure pipe will detect and execute
  }

  trackMouse() {
    // Any change detection will trigger impure pipe
    this.mousePosition++;
  }
}

// Optimized impure pipe with caching
@Pipe({
  name: 'filterOptimized',
  standalone: true,
  pure: false
})
export class FilterOptimizedPipe implements PipeTransform {
  private lastItems: any[] = [];
  private lastFilterField: string = '';
  private lastFilterValue: any;
  private cachedResult: any[] = [];

  transform(items: any[], filterField: string, filterValue: any): any[] {
    // Check if inputs actually changed
    if (
      items === this.lastItems &&
      filterField === this.lastFilterField &&
      filterValue === this.lastFilterValue
    ) {
      return this.cachedResult;
    }

    // Inputs changed, recalculate
    this.lastItems = items;
    this.lastFilterField = filterField;
    this.lastFilterValue = filterValue;
    
    this.cachedResult = items.filter(item => item[filterField] === filterValue);
    return this.cachedResult;
  }
}
```

**Performance Impact:**

Pure Pipes:
- Execute only when input reference changes
- Highly performant
- Safe for OnPush change detection
- Recommended for most use cases
- Require immutable data patterns

Impure Pipes:
- Execute on every change detection cycle
- Can cause performance issues
- Execute even when unrelated components change
- Should be used sparingly
- Useful for async operations or detecting deep changes

**Real-world Example:**

```typescript
// Date formatting (pure) - good performance
@Pipe({ name: 'customDate', standalone: true, pure: true })
export class CustomDatePipe implements PipeTransform {
  transform(value: Date, format: string = 'short'): string {
    // Executes only when date reference changes
    return formatDate(value, format, 'en-US');
  }
}

// Async pipe (impure) - necessary for observables
@Pipe({ name: 'asyncCustom', standalone: true, pure: false })
export class AsyncCustomPipe implements PipeTransform, OnDestroy {
  private subscription: Subscription | null = null;
  private latestValue: any = null;

  transform(observable: Observable<any>): any {
    if (this.subscription) {
      return this.latestValue;
    }

    this.subscription = observable.subscribe(value => {
      this.latestValue = value;
    });

    return this.latestValue;
  }

  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}
```

**Follow-up Questions:**
1. How can you optimize an impure pipe?
2. When is it acceptable to use an impure pipe?
3. How does the async pipe work internally?

**What Interviewers Look For:**
- Understanding of change detection impact
- Knowledge of when to use each type
- Awareness of performance implications
- Strategies for optimizing impure pipes

### 4. Explain the async pipe. How does it handle subscriptions and memory leaks?

**Expected Answer:**

The async pipe automatically subscribes to observables/promises and unsubscribes when the component is destroyed, preventing memory leaks.

**Implementation Details:**

```typescript
// How async pipe works internally (simplified)
@Pipe({
  name: 'asyncSimplified',
  standalone: true,
  pure: false // Must be impure to detect new values
})
export class AsyncSimplifiedPipe implements PipeTransform, OnDestroy {
  private subscription: Subscription | null = null;
  private latestValue: any = null;
  private latestReturnedValue: any = null;
  private observable: Observable<any> | null = null;
  private changeDetectorRef: ChangeDetectorRef;

  constructor(changeDetectorRef: ChangeDetectorRef) {
    this.changeDetectorRef = changeDetectorRef;
  }

  transform(observable: Observable<any> | null | undefined): any {
    if (!observable) {
      this.dispose();
      return null;
    }

    // If observable changed, subscribe to new one
    if (observable !== this.observable) {
      this.dispose();
      this.observable = observable;
      this.subscribe();
    }

    // Return latest value
    if (this.latestValue !== this.latestReturnedValue) {
      this.latestReturnedValue = this.latestValue;
      return this.latestValue;
    }

    return this.latestReturnedValue;
  }

  private subscribe() {
    if (this.observable) {
      this.subscription = this.observable.subscribe({
        next: (value) => {
          this.latestValue = value;
          // Trigger change detection when new value arrives
          this.changeDetectorRef.markForCheck();
        },
        error: (error) => {
          console.error('Async pipe error:', error);
        }
      });
    }
  }

  private dispose() {
    if (this.subscription) {
      this.subscription.unsubscribe();
      this.subscription = null;
    }
    this.latestValue = null;
    this.latestReturnedValue = null;
    this.observable = null;
  }

  ngOnDestroy() {
    this.dispose();
  }
}

// Practical usage examples
@Component({
  selector: 'app-async-demo',
  standalone: true,
  imports: [AsyncPipe, CommonModule],
  template: `
    <!-- Basic usage -->
    <div>{{ user$ | async }}</div>

    <!-- Multiple subscriptions to same observable (inefficient) -->
    <div>Name: {{ user$ | async.name }}</div>
    <div>Email: {{ user$ | async.email }}</div>

    <!-- Optimized: subscribe once, use as variable -->
    <div *ngIf="user$ | async as user">
      <div>Name: {{ user.name }}</div>
      <div>Email: {{ user.email }}</div>
      <div>Role: {{ user.role }}</div>
    </div>

    <!-- With loading and error states -->
    <ng-container *ngIf="userState$ | async as state">
      <div *ngIf="state.loading">Loading...</div>
      <div *ngIf="state.error">Error: {{ state.error }}</div>
      <div *ngIf="state.data">User: {{ state.data.name }}</div>
    </ng-container>

    <!-- Multiple async observables -->
    <div *ngIf="{
      user: user$ | async,
      posts: posts$ | async,
      comments: comments$ | async
    } as data">
      <app-user-profile [user]="data.user"></app-user-profile>
      <app-posts-list [posts]="data.posts"></app-posts-list>
      <app-comments-list [comments]="data.comments"></app-comments-list>
    </div>
  `
})
export class AsyncDemoComponent {
  user$ = this.http.get<User>('/api/user');
  posts$ = this.http.get<Post[]>('/api/posts');
  comments$ = this.http.get<Comment[]>('/api/comments');

  // Combined state pattern
  userState$ = this.http.get<User>('/api/user').pipe(
    map(data => ({ data, loading: false, error: null })),
    startWith({ data: null, loading: true, error: null }),
    catchError(error => of({ data: null, loading: false, error: error.message }))
  );

  constructor(private http: HttpClient) {}
}

// Advanced: Custom async pipe with retry logic
@Pipe({
  name: 'asyncRetry',
  standalone: true,
  pure: false
})
export class AsyncRetryPipe implements PipeTransform, OnDestroy {
  private subscription: Subscription | null = null;
  private latestValue: any = null;
  private observable: Observable<any> | null = null;
  private retryCount = 0;
  private maxRetries = 3;

  constructor(private changeDetectorRef: ChangeDetectorRef) {}

  transform(
    observable: Observable<any> | null,
    maxRetries: number = 3
  ): any {
    this.maxRetries = maxRetries;

    if (!observable) {
      this.dispose();
      return null;
    }

    if (observable !== this.observable) {
      this.dispose();
      this.observable = observable;
      this.subscribe();
    }

    return this.latestValue;
  }

  private subscribe() {
    if (this.observable) {
      this.subscription = this.observable.pipe(
        retry({
          count: this.maxRetries,
          delay: (error, retryCount) => {
            this.retryCount = retryCount;
            console.log(`Retry attempt ${retryCount}/${this.maxRetries}`);
            return timer(1000 * retryCount); // Exponential backoff
          }
        })
      ).subscribe({
        next: (value) => {
          this.latestValue = value;
          this.retryCount = 0;
          this.changeDetectorRef.markForCheck();
        },
        error: (error) => {
          this.latestValue = { error: error.message };
          this.changeDetectorRef.markForCheck();
        }
      });
    }
  }

  private dispose() {
    this.subscription?.unsubscribe();
    this.subscription = null;
    this.observable = null;
  }

  ngOnDestroy() {
    this.dispose();
  }
}
```

**Memory Management:**

```typescript
// BAD: Manual subscription can cause memory leak
@Component({
  selector: 'app-bad-subscription',
  template: `<div>{{ userName }}</div>`
})
export class BadSubscriptionComponent implements OnInit {
  userName: string = '';

  constructor(private userService: UserService) {}

  ngOnInit() {
    // Memory leak - no unsubscribe!
    this.userService.getUser().subscribe(user => {
      this.userName = user.name;
    });
  }
}

// GOOD: Using async pipe
@Component({
  selector: 'app-good-subscription',
  standalone: true,
  imports: [AsyncPipe],
  template: `<div>{{ (user$ | async)?.name }}</div>`
})
export class GoodSubscriptionComponent {
  user$ = this.userService.getUser();

  constructor(private userService: UserService) {}
  // No ngOnDestroy needed - async pipe handles cleanup
}

// ALTERNATIVE: Manual subscription with cleanup
@Component({
  selector: 'app-manual-subscription',
  template: `<div>{{ userName }}</div>`
})
export class ManualSubscriptionComponent implements OnInit, OnDestroy {
  userName: string = '';
  private subscription: Subscription = new Subscription();

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.subscription.add(
      this.userService.getUser().subscribe(user => {
        this.userName = user.name;
      })
    );
  }

  ngOnDestroy() {
    this.subscription.unsubscribe(); // Proper cleanup
  }
}
```

**Follow-up Questions:**
1. Why is the async pipe impure?
2. How does async pipe trigger change detection?
3. What happens if you use async pipe with the same observable multiple times?

**What Interviewers Look For:**
- Understanding of automatic subscription management
- Knowledge of memory leak prevention
- Best practices for using async pipe
- Awareness of performance implications with multiple async pipes

### 5. How do you create a custom pipe with parameters? Provide an example with multiple parameters.

**Expected Answer:**

Custom pipes can accept multiple parameters through the transform method's additional arguments.

**Comprehensive Examples:**

```typescript
// Simple custom pipe with single parameter
@Pipe({
  name: 'truncate',
  standalone: true,
  pure: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 50, ellipsis: string = '...'): string {
    if (!value) return '';
    if (value.length <= limit) return value;
    
    return value.substring(0, limit) + ellipsis;
  }
}

// Usage: {{ longText | truncate:100:'...' }}

// Advanced pipe with multiple parameters and options
@Pipe({
  name: 'formatCurrency',
  standalone: true,
  pure: true
})
export class FormatCurrencyPipe implements PipeTransform {
  transform(
    value: number,
    currencyCode: string = 'USD',
    display: 'code' | 'symbol' | 'name' = 'symbol',
    digitsInfo: string = '1.2-2',
    locale: string = 'en-US'
  ): string {
    if (value == null) return '';

    const formatter = new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: currencyCode,
      minimumFractionDigits: parseInt(digitsInfo.split('-')[0].split('.')[1]),
      maximumFractionDigits: parseInt(digitsInfo.split('-')[1])
    });

    return formatter.format(value);
  }
}

// Usage: {{ price | formatCurrency:'EUR':'symbol':'1.2-2':'de-DE' }}

// Pipe with object parameter for complex configuration
interface HighlightConfig {
  searchTerm: string;
  caseSensitive?: boolean;
  highlightClass?: string;
  wholeWord?: boolean;
}

@Pipe({
  name: 'highlight',
  standalone: true,
  pure: true
})
export class HighlightPipe implements PipeTransform {
  transform(value: string, config: HighlightConfig): string {
    if (!value || !config.searchTerm) return value;

    const {
      searchTerm,
      caseSensitive = false,
      highlightClass = 'highlight',
      wholeWord = false
    } = config;

    let flags = caseSensitive ? 'g' : 'gi';
    let pattern = wholeWord ? `\\b${searchTerm}\\b` : searchTerm;
    
    const regex = new RegExp(pattern, flags);
    
    return value.replace(regex, match => 
      `<span class="${highlightClass}">${match}</span>`
    );
  }
}

// Usage: 
// {{ text | highlight:{ searchTerm: 'angular', highlightClass: 'yellow-bg' } }}

// Chainable pipe with array manipulation
@Pipe({
  name: 'sort',
  standalone: true,
  pure: true
})
export class SortPipe implements PipeTransform {
  transform<T>(
    array: T[],
    field: keyof T,
    order: 'asc' | 'desc' = 'asc',
    caseInsensitive: boolean = true
  ): T[] {
    if (!array || !field) return array;

    return [...array].sort((a, b) => {
      let valueA = a[field];
      let valueB = b[field];

      // Handle case insensitive string comparison
      if (caseInsensitive && typeof valueA === 'string' && typeof valueB === 'string') {
        valueA = valueA.toLowerCase() as any;
        valueB = valueB.toLowerCase() as any;
      }

      if (valueA < valueB) return order === 'asc' ? -1 : 1;
      if (valueA > valueB) return order === 'asc' ? 1 : -1;
      return 0;
    });
  }
}

// Usage: {{ users | sort:'name':'asc':true }}

// Pipe with dependency injection
@Pipe({
  name: 'translate',
  standalone: true,
  pure: false // Impure to react to language changes
})
export class TranslatePipe implements PipeTransform {
  constructor(private translateService: TranslateService) {}

  transform(key: string, params?: Record<string, any>): string {
    return this.translateService.instant(key, params);
  }
}

// Usage: {{ 'WELCOME_MESSAGE' | translate:{ name: userName } }}

// Complex pipe with async operations and caching
@Pipe({
  name: 'userLookup',
  standalone: true,
  pure: true
})
export class UserLookupPipe implements PipeTransform {
  private cache = new Map<string, string>();

  constructor(private userService: UserService) {}

  transform(userId: string, field: 'name' | 'email' = 'name'): Observable<string> {
    const cacheKey = `${userId}-${field}`;

    if (this.cache.has(cacheKey)) {
      return of(this.cache.get(cacheKey)!);
    }

    return this.userService.getUser(userId).pipe(
      map(user => {
        const value = user[field];
        this.cache.set(cacheKey, value);
        return value;
      }),
      catchError(() => of('Unknown'))
    );
  }
}

// Usage: {{ userId | userLookup:'name' | async }}

// Comprehensive example in a component
@Component({
  selector: 'app-pipe-showcase',
  standalone: true,
  imports: [
    CommonModule,
    TruncatePipe,
    FormatCurrencyPipe,
    HighlightPipe,
    SortPipe,
    TranslatePipe,
    UserLookupPipe,
    AsyncPipe
  ],
  template: `
    <!-- Truncate with custom parameters -->
    <p>{{ description | truncate:150:'... Read more' }}</p>

    <!-- Currency formatting -->
    <p>Price: {{ price | formatCurrency:'EUR':'symbol':'1.2-2':'de-DE' }}</p>

    <!-- Highlighting with configuration object -->
    <div [innerHTML]="text | highlight:highlightConfig"></div>

    <!-- Sorting with multiple parameters -->
    <div *ngFor="let user of users | sort:'lastName':'asc':true">
      {{ user.lastName }}, {{ user.firstName }}
    </div>

    <!-- Chaining pipes -->
    <div *ngFor="let item of items | sort:'price':'desc' | slice:0:5">
      {{ item.name }}: {{ item.price | formatCurrency }}
    </div>

    <!-- Translation with parameters -->
    <h1>{{ 'GREETING' | translate:{ name: currentUser.name } }}</h1>

    <!-- Async pipe with custom pipe -->
    <p>Created by: {{ creatorId | userLookup:'name' | async }}</p>
  `
})
export class PipeShowcaseComponent {
  description = 'Very long description text that needs to be truncated...';
  price = 1234.56;
  text = 'Angular is a powerful framework for building web applications';
  highlightConfig = { searchTerm: 'Angular', highlightClass: 'highlight-yellow' };
  
  users = [
    { firstName: 'John', lastName: 'Doe' },
    { firstName: 'Jane', lastName: 'Smith' }
  ];

  items = [
    { name: 'Item 1', price: 100 },
    { name: 'Item 2', price: 200 }
  ];

  currentUser = { name: 'John' };
  creatorId = 'user-123';
}
```

**Follow-up Questions:**
1. How do you test custom pipes with parameters?
2. What's the performance impact of using multiple pipes in a chain?
3. How do you handle errors in pipes?

**What Interviewers Look For:**
- Proper use of transform method parameters
- Understanding of pure vs impure in context of parameters
- Knowledge of pipe chaining
- Proper typing with TypeScript generics

## Advanced Directive Patterns

### 6. How do you create a directive that communicates with its host component?

**Expected Answer:**

Directives can communicate with host components through @Output events, services, or by directly injecting the host component.

**Implementation:**

```typescript
// Method 1: Using @Output events
@Directive({
  selector: '[appClickTracker]',
  standalone: true
})
export class ClickTrackerDirective {
  @Output() clicked = new EventEmitter<ClickEvent>();
  private clickCount = 0;

  @HostListener('click', ['$event'])
  onClick(event: MouseEvent) {
    this.clickCount++;
    this.clicked.emit({
      count: this.clickCount,
      timestamp: new Date(),
      target: event.target
    });
  }
}

// Method 2: Using component injection
@Directive({
  selector: '[appFormField]',
  standalone: true
})
export class FormFieldDirective implements OnInit {
  @Input() fieldName: string = '';

  constructor(
    @Optional() @Host() private formComponent: FormComponent,
    private elementRef: ElementRef
  ) {}

  ngOnInit() {
    if (this.formComponent) {
      this.formComponent.registerField(this.fieldName, this.elementRef);
    }
  }
}

// Method 3: Using shared service
@Injectable()
export class SelectionService {
  private selectedItems = new Set<string>();
  selectedItems$ = new BehaviorSubject<Set<string>>(this.selectedItems);

  toggle(id: string) {
    if (this.selectedItems.has(id)) {
      this.selectedItems.delete(id);
    } else {
      this.selectedItems.add(id);
    }
    this.selectedItems$.next(new Set(this.selectedItems));
  }
}

@Directive({
  selector: '[appSelectable]',
  standalone: true
})
export class SelectableDirective {
  @Input() appSelectable: string = '';

  constructor(private selectionService: SelectionService) {}

  @HostListener('click')
  onClick() {
    this.selectionService.toggle(this.appSelectable);
  }
}
```

## Performance Considerations

### 7. What are the performance implications of using pipes vs functions in templates?

**Expected Answer:**

Pipes are cached and only recalculate when inputs change (pure pipes), while functions execute on every change detection cycle.

**Comparison:**

```typescript
@Component({
  selector: 'app-performance-demo',
  standalone: true,
  imports: [FilterPipe],
  template: `
    <!-- BAD: Function called on every change detection -->
    <div *ngFor="let item of filterItems(items, 'active')">
      {{ item.name }}
    </div>

    <!-- GOOD: Pure pipe called only when inputs change -->
    <div *ngFor="let item of items | filter:'active'">
      {{ item.name }}
    </div>

    <!-- BETTER: Filter in component, use OnPush -->
    <div *ngFor="let item of filteredItems">
      {{ item.name }}
    </div>
  `
})
export class PerformanceDemoComponent {
  items = [...];

  // Called on EVERY change detection - expensive!
  filterItems(items: any[], status: string) {
    console.log('Function called');
    return items.filter(item => item.status === status);
  }

  // Calculated once, cached
  get filteredItems() {
    return this.items.filter(item => item.status === 'active');
  }
}
```

## Key Takeaways

1. **Structural vs Attribute Directives**: Structural directives modify DOM structure using TemplateRef and ViewContainerRef, while attribute directives change element behavior or appearance without altering DOM structure.

2. **TemplateRef and ViewContainerRef**: Essential for custom structural directives - TemplateRef represents the template content, ViewContainerRef is the container that manages views.

3. **Pure vs Impure Pipes**: Pure pipes (default) execute only when input references change, making them performant. Impure pipes execute on every change detection cycle and should be used sparingly.

4. **Async Pipe Benefits**: Automatically handles subscription lifecycle, prevents memory leaks, and works seamlessly with OnPush change detection strategy.

5. **Custom Pipe Parameters**: Pass multiple parameters through transform method arguments. Use object parameters for complex configurations.

6. **Directive Communication**: Use @Output events, component injection with @Host/@Optional, or shared services for directive-component communication.

7. **Memory Management**: Always clean up subscriptions in directives. Use ngOnDestroy for proper cleanup. The async pipe handles this automatically.

8. **Performance Optimization**: Prefer pipes over template functions. Cache results in impure pipes. Use pure pipes with immutable data patterns.

9. **Context Variables**: Structural directives can expose context variables using $implicit and named properties for use in templates.

10. **Testing Strategy**: Test directives in isolation using TestBed. Test pipes as pure functions. Mock dependencies appropriately and verify DOM changes.

## Red Flags to Avoid

- Creating impure pipes without good reason or caching strategy
- Using template functions instead of pipes for transformations
- Not cleaning up subscriptions in custom directives
- Mutating input arrays in pipes (always return new arrays)
- Creating structural directives without understanding ViewContainerRef
- Using async pipe multiple times on the same observable
- Not handling null/undefined in pipe transform methods
- Forgetting to mark directive inputs with @Input
- Not using HostListener/HostBinding for DOM interactions
- Creating overly complex pipes that should be component logic

## Interview Preparation Tips

- Practice creating both structural and attribute directives
- Understand the async pipe implementation deeply
- Be ready to explain pure vs impure with performance examples
- Know how to debug directive issues using Angular DevTools
- Prepare examples of real-world custom pipes you've created
- Understand pipe chaining and operator precedence
- Know when to use pipes vs getters vs component properties
- Be familiar with built-in pipes and their use cases
