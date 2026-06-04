# Custom Structural Directives

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Structural Directives](#understanding-structural-directives)
3. [TemplateRef and ViewContainerRef](#templateref-and-viewcontainerref)
4. [Creating Basic Structural Directives](#creating-basic-structural-directives)
5. [Microsyntax and Context Objects](#microsyntax-and-context-objects)
6. [Advanced Structural Directive Patterns](#advanced-structural-directive-patterns)
7. [Multiple Template Contexts](#multiple-template-contexts)
8. [Performance Considerations](#performance-considerations)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)
12. [Key Takeaways](#key-takeaways)
13. [Resources](#resources)

## Introduction

Structural directives are one of Angular's most powerful features, allowing you to manipulate the DOM structure by adding, removing, or replacing elements. While Angular provides built-in structural directives like `*ngIf`, `*ngFor`, and `*ngSwitch`, creating custom structural directives enables you to encapsulate complex DOM manipulation logic and create reusable template patterns.

This guide covers everything you need to know about building custom structural directives, from basic concepts to advanced patterns with microsyntax and context objects.

## Understanding Structural Directives

Structural directives change the DOM layout by adding or removing DOM elements. They're identified by the asterisk (*) prefix, which is syntactic sugar that Angular transforms into a more verbose form.

### How Angular Transforms Structural Directives

```typescript
// What you write
<div *ngIf="condition">Content</div>

// What Angular transforms it into
<ng-template [ngIf]="condition">
  <div>Content</div>
</ng-template>
```

### The Three Key Classes

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef 
} from '@angular/core';

@Directive({
  selector: '[appCustomStructural]',
  standalone: true
})
export class CustomStructuralDirective {
  constructor(
    private templateRef: TemplateRef<any>,      // Reference to the template
    private viewContainerRef: ViewContainerRef  // Container where view is rendered
  ) {}
}
```

## TemplateRef and ViewContainerRef

### TemplateRef

`TemplateRef` represents an embedded template that can be used to instantiate embedded views.

```typescript
import { Directive, TemplateRef } from '@angular/core';

@Directive({
  selector: '[appTemplateExample]',
  standalone: true
})
export class TemplateExampleDirective {
  constructor(private template: TemplateRef<any>) {
    console.log('Template:', this.template);
    // TemplateRef provides access to the template content
    // but doesn't render it directly
  }
}
```

### ViewContainerRef

`ViewContainerRef` represents a container where one or more views can be attached.

```typescript
import { Directive, ViewContainerRef, TemplateRef, OnInit } from '@angular/core';

@Directive({
  selector: '[appViewContainerExample]',
  standalone: true
})
export class ViewContainerExampleDirective implements OnInit {
  constructor(
    private viewContainer: ViewContainerRef,
    private template: TemplateRef<any>
  ) {}

  ngOnInit() {
    // Create and attach a view from the template
    this.viewContainer.createEmbeddedView(this.template);
  }
}
```

### ViewContainerRef Methods

```typescript
import { Directive, ViewContainerRef, TemplateRef } from '@angular/core';

@Directive({
  selector: '[appViewMethods]',
  standalone: true
})
export class ViewMethodsDirective {
  constructor(
    private viewContainer: ViewContainerRef,
    private template: TemplateRef<any>
  ) {}

  demonstrateMethods() {
    // Create embedded view
    const viewRef = this.viewContainer.createEmbeddedView(this.template);
    
    // Get view count
    const count = this.viewContainer.length;
    
    // Get view at index
    const view = this.viewContainer.get(0);
    
    // Insert view at specific index
    this.viewContainer.insert(viewRef, 0);
    
    // Move view to different index
    this.viewContainer.move(viewRef, 1);
    
    // Remove view at index
    this.viewContainer.remove(0);
    
    // Detach view (doesn't destroy it)
    this.viewContainer.detach(0);
    
    // Clear all views
    this.viewContainer.clear();
  }
}
```

## Creating Basic Structural Directives

### Simple Conditional Directive

```typescript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true
})
export class UnlessDirective {
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      // Create view when condition is false
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      // Remove view when condition is true
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [UnlessDirective],
  template: `
    <div *appUnless="isHidden">
      This content shows when isHidden is false
    </div>
  `
})
export class AppComponent {
  isHidden = false;
}
```

### Repeat Directive

```typescript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appRepeat]',
  standalone: true
})
export class RepeatDirective {
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appRepeat(times: number) {
    this.viewContainer.clear();
    
    for (let i = 0; i < times; i++) {
      this.viewContainer.createEmbeddedView(this.templateRef, {
        $implicit: i,
        index: i,
        count: times
      });
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RepeatDirective],
  template: `
    <div *appRepeat="3; let i">
      Iteration {{ i }}
    </div>
  `
})
export class AppComponent {}
```

### Delay Directive

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  OnInit,
  OnDestroy 
} from '@angular/core';

@Directive({
  selector: '[appDelay]',
  standalone: true
})
export class DelayDirective implements OnInit, OnDestroy {
  @Input() appDelay = 0;
  private timeoutId?: number;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  ngOnInit() {
    this.timeoutId = window.setTimeout(() => {
      this.viewContainer.createEmbeddedView(this.templateRef);
    }, this.appDelay);
  }

  ngOnDestroy() {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [DelayDirective],
  template: `
    <div *appDelay="2000">
      This appears after 2 seconds
    </div>
  `
})
export class AppComponent {}
```

## Microsyntax and Context Objects

### Understanding Microsyntax

Microsyntax is the special syntax used with structural directives that Angular parses and transforms.

```typescript
// Microsyntax patterns
*directive="expression"
*directive="expression; let variable"
*directive="expression as localVar"
*directive="let item of items; let i = index"
```

### Context Objects

Context objects provide data to the template through local template variables.

```typescript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

interface RangeContext {
  $implicit: number;  // Default variable (let item)
  index: number;      // let i = index
  first: boolean;     // let isFirst = first
  last: boolean;      // let isLast = last
  even: boolean;      // let isEven = even
  odd: boolean;       // let isOdd = odd
}

@Directive({
  selector: '[appRange]',
  standalone: true
})
export class RangeDirective {
  constructor(
    private templateRef: TemplateRef<RangeContext>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appRange(value: number | { from: number; to: number }) {
    this.viewContainer.clear();

    let start = 0;
    let end = 0;

    if (typeof value === 'number') {
      end = value;
    } else {
      start = value.from;
      end = value.to;
    }

    const length = end - start;
    
    for (let i = 0; i < length; i++) {
      const context: RangeContext = {
        $implicit: start + i,
        index: i,
        first: i === 0,
        last: i === length - 1,
        even: i % 2 === 0,
        odd: i % 2 !== 0
      };
      
      this.viewContainer.createEmbeddedView(this.templateRef, context);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RangeDirective],
  template: `
    <!-- Using default $implicit -->
    <div *appRange="5; let num">{{ num }}</div>
    
    <!-- Using all context variables -->
    <div *appRange="5; let num; let i = index; let isFirst = first; let isLast = last">
      {{ num }} - Index: {{ i }} - First: {{ isFirst }} - Last: {{ isLast }}
    </div>
    
    <!-- Using from/to syntax -->
    <div *appRange="{ from: 10, to: 15 }; let num">{{ num }}</div>
  `
})
export class AppComponent {}
```

### Complex Microsyntax Example

```typescript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

interface ForOfContext<T> {
  $implicit: T;
  appForOf: T[];
  index: number;
  count: number;
  first: boolean;
  last: boolean;
  even: boolean;
  odd: boolean;
}

@Directive({
  selector: '[appForOf]',
  standalone: true
})
export class ForOfDirective<T> {
  constructor(
    private templateRef: TemplateRef<ForOfContext<T>>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appForOf(items: T[]) {
    this.viewContainer.clear();

    items.forEach((item, index) => {
      const context: ForOfContext<T> = {
        $implicit: item,
        appForOf: items,
        index,
        count: items.length,
        first: index === 0,
        last: index === items.length - 1,
        even: index % 2 === 0,
        odd: index % 2 !== 0
      };

      this.viewContainer.createEmbeddedView(this.templateRef, context);
    });
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ForOfDirective],
  template: `
    <div *appForOf="let item of items; let i = index; let isFirst = first">
      <span [class.highlight]="isFirst">
        {{ i }}: {{ item }}
      </span>
    </div>
  `
})
export class AppComponent {
  items = ['Apple', 'Banana', 'Cherry'];
}
```

## Advanced Structural Directive Patterns

### Permission-Based Rendering

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  OnInit,
  inject
} from '@angular/core';

export interface User {
  id: string;
  roles: string[];
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUser = signal<User | null>(null);

  hasPermission(roles: string[]): boolean {
    const user = this.currentUser();
    if (!user) return false;
    return roles.some(role => user.roles.includes(role));
  }
}

@Directive({
  selector: '[appHasRole]',
  standalone: true
})
export class HasRoleDirective implements OnInit {
  @Input() appHasRole: string | string[] = [];
  @Input() appHasRoleElse?: TemplateRef<any>;

  private authService = inject(AuthService);
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  ngOnInit() {
    const roles = Array.isArray(this.appHasRole) 
      ? this.appHasRole 
      : [this.appHasRole];

    if (this.authService.hasPermission(roles)) {
      if (!this.hasView) {
        this.viewContainer.createEmbeddedView(this.templateRef);
        this.hasView = true;
      }
    } else if (this.appHasRoleElse) {
      this.viewContainer.createEmbeddedView(this.appHasRoleElse);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HasRoleDirective],
  template: `
    <div *appHasRole="'admin'">
      Admin content
    </div>

    <div *appHasRole="['admin', 'editor']; else noAccess">
      Admin or Editor content
    </div>

    <ng-template #noAccess>
      <div>You don't have access</div>
    </ng-template>
  `
})
export class AppComponent {}
```

### Loading State Directive

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  OnChanges 
} from '@angular/core';

interface LoadingContext<T> {
  $implicit: T;
  appLoading: boolean;
  appLoadingData: T;
}

@Directive({
  selector: '[appLoading]',
  standalone: true
})
export class LoadingDirective<T> implements OnChanges {
  @Input() appLoading = false;
  @Input() appLoadingData?: T;
  @Input() appLoadingTemplate?: TemplateRef<any>;

  constructor(
    private templateRef: TemplateRef<LoadingContext<T>>,
    private viewContainer: ViewContainerRef
  ) {}

  ngOnChanges() {
    this.viewContainer.clear();

    if (this.appLoading && this.appLoadingTemplate) {
      // Show loading template
      this.viewContainer.createEmbeddedView(this.appLoadingTemplate);
    } else if (!this.appLoading && this.appLoadingData !== undefined) {
      // Show content with data
      const context: LoadingContext<T> = {
        $implicit: this.appLoadingData,
        appLoading: false,
        appLoadingData: this.appLoadingData
      };
      this.viewContainer.createEmbeddedView(this.templateRef, context);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [LoadingDirective],
  template: `
    <div *appLoading="isLoading; data: userData; template: loading; let user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>

    <ng-template #loading>
      <div class="spinner">Loading...</div>
    </ng-template>
  `
})
export class AppComponent {
  isLoading = true;
  userData?: { name: string; email: string };

  ngOnInit() {
    setTimeout(() => {
      this.userData = { name: 'John', email: 'john@example.com' };
      this.isLoading = false;
    }, 2000);
  }
}
```

### Viewport Visibility Directive

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  OnInit,
  OnDestroy,
  ElementRef 
} from '@angular/core';

@Directive({
  selector: '[appInViewport]',
  standalone: true
})
export class InViewportDirective implements OnInit, OnDestroy {
  @Input() appInViewportRootMargin = '0px';
  @Input() appInViewportThreshold = 0.5;

  private observer?: IntersectionObserver;
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private element: ElementRef
  ) {}

  ngOnInit() {
    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && !this.hasView) {
          this.viewContainer.createEmbeddedView(this.templateRef);
          this.hasView = true;
          // Optionally disconnect after first render
          // this.observer?.disconnect();
        } else if (!entry.isIntersecting && this.hasView) {
          this.viewContainer.clear();
          this.hasView = false;
        }
      },
      {
        rootMargin: this.appInViewportRootMargin,
        threshold: this.appInViewportThreshold
      }
    );

    // Create a placeholder element to observe
    const placeholder = document.createElement('div');
    this.viewContainer.element.nativeElement.parentNode?.insertBefore(
      placeholder,
      this.viewContainer.element.nativeElement
    );
    this.observer.observe(placeholder);
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [InViewportDirective],
  template: `
    <div style="height: 200vh;">Scroll down</div>
    
    <div *appInViewport="{ threshold: 0.5, rootMargin: '100px' }">
      <img src="large-image.jpg" alt="Lazy loaded">
    </div>
  `
})
export class AppComponent {}
```

## Multiple Template Contexts

### If-Then-Else Directive

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef 
} from '@angular/core';

@Directive({
  selector: '[appIfThenElse]',
  standalone: true
})
export class IfThenElseDirective {
  private condition = false;
  private thenTemplate?: TemplateRef<any>;
  private elseTemplate?: TemplateRef<any>;

  constructor(
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appIfThenElse(condition: boolean) {
    this.condition = condition;
    this.updateView();
  }

  @Input() set appIfThenElseThen(template: TemplateRef<any>) {
    this.thenTemplate = template;
    this.updateView();
  }

  @Input() set appIfThenElseElse(template: TemplateRef<any>) {
    this.elseTemplate = template;
    this.updateView();
  }

  private updateView() {
    this.viewContainer.clear();

    if (this.condition && this.thenTemplate) {
      this.viewContainer.createEmbeddedView(this.thenTemplate);
    } else if (!this.condition && this.elseTemplate) {
      this.viewContainer.createEmbeddedView(this.elseTemplate);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [IfThenElseDirective],
  template: `
    <div *appIfThenElse="isLoggedIn; then: loggedIn; else: loggedOut"></div>

    <ng-template #loggedIn>
      <h2>Welcome back!</h2>
    </ng-template>

    <ng-template #loggedOut>
      <button>Please log in</button>
    </ng-template>
  `
})
export class AppComponent {
  isLoggedIn = false;
}
```

### Switch-Case Directive

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  ContentChildren,
  QueryList,
  AfterContentInit 
} from '@angular/core';

@Directive({
  selector: '[appSwitchCase]',
  standalone: true
})
export class SwitchCaseDirective {
  @Input() appSwitchCase: any;

  constructor(public template: TemplateRef<any>) {}
}

@Directive({
  selector: '[appSwitchDefault]',
  standalone: true
})
export class SwitchDefaultDirective {
  constructor(public template: TemplateRef<any>) {}
}

@Directive({
  selector: '[appSwitch]',
  standalone: true
})
export class SwitchDirective implements AfterContentInit {
  @Input() appSwitch: any;
  @ContentChildren(SwitchCaseDirective) cases!: QueryList<SwitchCaseDirective>;
  @ContentChildren(SwitchDefaultDirective) defaultCase!: QueryList<SwitchDefaultDirective>;

  constructor(private viewContainer: ViewContainerRef) {}

  ngAfterContentInit() {
    this.updateView();
  }

  @Input() set appSwitchValue(value: any) {
    this.appSwitch = value;
    this.updateView();
  }

  private updateView() {
    this.viewContainer.clear();

    const matchingCase = this.cases?.find(
      c => c.appSwitchCase === this.appSwitch
    );

    if (matchingCase) {
      this.viewContainer.createEmbeddedView(matchingCase.template);
    } else if (this.defaultCase?.first) {
      this.viewContainer.createEmbeddedView(this.defaultCase.first.template);
    }
  }
}
```

## Performance Considerations

### Optimizing View Creation

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef,
  EmbeddedViewRef 
} from '@angular/core';

@Directive({
  selector: '[appOptimizedList]',
  standalone: true
})
export class OptimizedListDirective<T> {
  private viewCache = new Map<T, EmbeddedViewRef<any>>();

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appOptimizedList(items: T[]) {
    // Track which items should be rendered
    const itemSet = new Set(items);
    
    // Remove views for items no longer in the list
    for (const [item, view] of this.viewCache) {
      if (!itemSet.has(item)) {
        const index = this.viewContainer.indexOf(view);
        if (index !== -1) {
          this.viewContainer.remove(index);
        }
        this.viewCache.delete(item);
      }
    }

    // Add or reorder views
    items.forEach((item, index) => {
      let view = this.viewCache.get(item);
      
      if (!view) {
        // Create new view
        view = this.viewContainer.createEmbeddedView(
          this.templateRef,
          { $implicit: item, index }
        );
        this.viewCache.set(item, view);
      } else {
        // Move existing view
        const currentIndex = this.viewContainer.indexOf(view);
        if (currentIndex !== index) {
          this.viewContainer.move(view, index);
        }
      }
    });
  }
}
```

### Lazy View Creation

```typescript
import { 
  Directive, 
  Input, 
  TemplateRef, 
  ViewContainerRef 
} from '@angular/core';

@Directive({
  selector: '[appLazyRender]',
  standalone: true
})
export class LazyRenderDirective {
  @Input() appLazyRenderThreshold = 100; // ms
  private timeoutId?: number;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appLazyRender(condition: boolean) {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }

    if (condition) {
      // Delay rendering to next frame or after threshold
      this.timeoutId = window.setTimeout(() => {
        this.viewContainer.createEmbeddedView(this.templateRef);
      }, this.appLazyRenderThreshold);
    } else {
      this.viewContainer.clear();
    }
  }
}
```

## Common Mistakes

### 1. Not Tracking View State

```typescript
// BAD: Creating views without tracking
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective {
  @Input() set appBad(condition: boolean) {
    if (condition) {
      // This creates a new view every time!
      this.viewContainer.createEmbeddedView(this.templateRef);
    }
  }
}

// GOOD: Track whether view exists
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective {
  private hasView = false;

  @Input() set appGood(condition: boolean) {
    if (condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (!condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
```

### 2. Incorrect Context Type

```typescript
// BAD: Loose typing
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective {
  constructor(private template: TemplateRef<any>) {}
}

// GOOD: Strict typing
interface MyContext {
  $implicit: string;
  index: number;
}

@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective {
  constructor(private template: TemplateRef<MyContext>) {}
}
```

### 3. Memory Leaks

```typescript
// BAD: Not cleaning up observers
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective implements OnInit {
  ngOnInit() {
    const observer = new IntersectionObserver(() => {});
    // Never disconnected!
  }
}

// GOOD: Proper cleanup
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective implements OnInit, OnDestroy {
  private observer?: IntersectionObserver;

  ngOnInit() {
    this.observer = new IntersectionObserver(() => {});
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}
```

### 4. Incorrect Microsyntax Naming

```typescript
// BAD: Inconsistent naming
@Directive({ selector: '[appMyDirective]', standalone: true })
export class BadDirective {
  @Input() value: any; // Doesn't match selector
}

// GOOD: Consistent naming for microsyntax
@Directive({ selector: '[appMyDirective]', standalone: true })
export class GoodDirective {
  @Input() appMyDirective: any; // Matches selector
  @Input() appMyDirectiveElse?: TemplateRef<any>; // Proper suffix
}
```

## Best Practices

### 1. Use Descriptive Context Interfaces

```typescript
interface UserListContext {
  $implicit: User;
  user: User;
  index: number;
  count: number;
  first: boolean;
  last: boolean;
}

@Directive({
  selector: '[appUserList]',
  standalone: true
})
export class UserListDirective {
  constructor(private template: TemplateRef<UserListContext>) {}
}
```

### 2. Provide Default Templates

```typescript
@Directive({
  selector: '[appLoadable]',
  standalone: true
})
export class LoadableDirective {
  @Input() appLoadableLoading?: TemplateRef<any>;
  @Input() appLoadableError?: TemplateRef<any>;

  private defaultLoadingTemplate = `<div>Loading...</div>`;
  private defaultErrorTemplate = `<div>Error occurred</div>`;
}
```

### 3. Use Change Detection Wisely

```typescript
import { ChangeDetectorRef } from '@angular/core';

@Directive({
  selector: '[appOptimized]',
  standalone: true
})
export class OptimizedDirective {
  constructor(
    private cdr: ChangeDetectorRef,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appOptimized(data: any) {
    this.viewContainer.clear();
    this.viewContainer.createEmbeddedView(this.template, { $implicit: data });
    // Manually trigger change detection if needed
    this.cdr.detectChanges();
  }
}
```

### 4. Document Microsyntax Usage

```typescript
/**
 * Renders content based on user permissions
 * 
 * @example
 * <div *appPermission="'admin'; else: denied">Admin content</div>
 * 
 * @example
 * <div *appPermission="['admin', 'editor']; let role">
 *   Current role: {{ role }}
 * </div>
 */
@Directive({
  selector: '[appPermission]',
  standalone: true
})
export class PermissionDirective {
  // Implementation
}
```

## Interview Questions

### Q1: What is the difference between TemplateRef and ViewContainerRef?

**Answer:** `TemplateRef` represents a template that can be used to create views, but doesn't render anything by itself. It's like a blueprint. `ViewContainerRef` is a container that can hold one or more views - it's where templates are actually rendered. Think of `TemplateRef` as the recipe and `ViewContainerRef` as the kitchen where you cook.

### Q2: How does Angular transform structural directive syntax?

**Answer:** Angular transforms the asterisk (*) syntax into an `<ng-template>` wrapper:
```typescript
*ngIf="condition" becomes <ng-template [ngIf]="condition">
```
This transformation allows Angular to work with the template without rendering it immediately.

### Q3: What is the $implicit property in context objects?

**Answer:** `$implicit` is a special property that provides the default value for template variables declared without an assignment. For example, in `let item`, the value comes from `$implicit`, while in `let i = index`, it comes from the `index` property.

### Q4: How do you prevent memory leaks in structural directives?

**Answer:** Implement `OnDestroy` and clean up subscriptions, observers, timers, and event listeners. Always disconnect IntersectionObservers, clear timeouts/intervals, and unsubscribe from Observables.

### Q5: Can you create multiple views from the same template?

**Answer:** Yes, you can call `createEmbeddedView()` multiple times with the same `TemplateRef` to create multiple instances. This is how `*ngFor` creates multiple elements from a single template.

### Q6: How do structural directives impact change detection?

**Answer:** Each embedded view created by a structural directive participates in change detection. When views are created or destroyed, Angular updates its change detection tree. Using `OnPush` change detection with structural directives can improve performance.

### Q7: What's the difference between detach() and remove() on ViewContainerRef?

**Answer:** `detach()` removes the view from the container but keeps it in memory, allowing you to reattach it later. `remove()` completely destroys the view and its associated data. Use `detach()` when you might need the view again, `remove()` when you're done with it.

### Q8: How do you create a structural directive with multiple inputs?

**Answer:** Use the directive selector as the base name and add suffixes for additional inputs:
```typescript
@Input() appMyDir: any;
@Input() appMyDirParam1: any;
@Input() appMyDirParam2: any;
```

## Key Takeaways

1. **Structural directives manipulate DOM structure** using `TemplateRef` and `ViewContainerRef`
2. **The asterisk (*) is syntactic sugar** that Angular transforms into `<ng-template>` wrappers
3. **TemplateRef is the template blueprint**, ViewContainerRef is where it renders
4. **Context objects provide data to templates** through the `$implicit` property and named properties
5. **Always track view state** to avoid creating duplicate views
6. **Use TypeScript interfaces** for context objects to ensure type safety
7. **Clean up resources in OnDestroy** to prevent memory leaks
8. **Microsyntax naming must be consistent** with the directive selector
9. **Cache and reuse views** when possible for better performance
10. **Document your directive's microsyntax** with examples for other developers

## Resources

### Official Documentation
- [Angular Structural Directives](https://angular.dev/guide/directives/structural-directives)
- [TemplateRef API](https://angular.dev/api/core/TemplateRef)
- [ViewContainerRef API](https://angular.dev/api/core/ViewContainerRef)

### Articles
- "Writing Structural Directives" - Angular University
- "Advanced Angular Directives" - Thoughtram Blog
- "Custom Structural Directives Deep Dive" - Netanel Basal

### Video Tutorials
- "Angular Structural Directives Explained" - Angular Connect
- "Creating Custom Directives" - ng-conf
- "Master Angular Directives" - Decoded Frontend

### Books
- "Angular Development with TypeScript" - Chapter on Directives
- "ng-book: The Complete Guide to Angular" - Directive Patterns

### Tools
- Angular DevTools - View directive lifecycle
- VS Code Angular Language Service - Directive intellisense
