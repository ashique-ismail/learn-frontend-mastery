# Angular Lifecycle Hooks

## The Idea

**In plain English:** Lifecycle hooks are special functions that Angular (the framework that builds your web app) automatically calls at specific moments during a component's life — like when it first appears on the screen, when its data changes, or when it disappears. A "component" is just a reusable piece of your webpage, like a button or a user card.

**Real-world analogy:** Think of opening and closing a pop-up lemonade stand. The stand goes through predictable stages every day, and at each stage you do specific tasks:

- The moment you unlock the stand and set up = `ngOnInit` (component first appears, run your setup code)
- Every time a customer places a new order = `ngOnChanges` (an input value from outside changes)
- Checking stock levels between every customer = `ngDoCheck` (Angular checking for any change it may have missed)
- When your helper arrives and arranges the extras (napkins, signs) = `ngAfterContentInit` (projected/external content is ready)
- Inspecting the extras after each customer = `ngAfterContentChecked` (projected content is re-checked)
- When the stand's own display board is fully set up = `ngAfterViewInit` (the component's own view and child views are ready)
- Reviewing the display board after every customer = `ngAfterViewChecked` (the view is re-checked each cycle)
- Packing everything up at closing time = `ngOnDestroy` (component is removed; clean up timers, subscriptions, etc.)

---

## Table of Contents

- [Introduction](#introduction)
- [Component Lifecycle Overview](#component-lifecycle-overview)
- [All Lifecycle Hooks](#all-lifecycle-hooks)
- [Hook Execution Order](#hook-execution-order)
- [Detailed Hook Explanations](#detailed-hook-explanations)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

Angular lifecycle hooks are methods that allow you to tap into key moments in a component or directive's lifecycle, from creation through destruction. Understanding these hooks is crucial for managing component initialization, responding to changes, cleaning up resources, and optimizing performance.

Each hook serves a specific purpose and executes at a precise moment in the component's lifecycle. Proper use of lifecycle hooks enables you to write more efficient, maintainable, and bug-free Angular applications.

## Component Lifecycle Overview

The Angular component lifecycle follows this general flow:

1. **Creation**: Constructor is called, dependency injection occurs
2. **Initialization**: Angular sets input properties and calls initialization hooks
3. **Change Detection**: Angular checks for changes and calls change detection hooks
4. **Content & View Projection**: Child components and projected content are initialized
5. **Updates**: Angular responds to data changes through change detection cycles
6. **Destruction**: Component is removed from the DOM, cleanup occurs

## All Lifecycle Hooks

Angular provides the following lifecycle hooks:

```typescript
interface OnChanges {
  ngOnChanges(changes: SimpleChanges): void;
}

interface OnInit {
  ngOnInit(): void;
}

interface DoCheck {
  ngDoCheck(): void;
}

interface AfterContentInit {
  ngAfterContentInit(): void;
}

interface AfterContentChecked {
  ngAfterContentChecked(): void;
}

interface AfterViewInit {
  ngAfterViewInit(): void;
}

interface AfterViewChecked {
  ngAfterViewChecked(): void;
}

interface OnDestroy {
  ngOnDestroy(): void;
}
```

## Hook Execution Order

During component initialization:

```
1. Constructor
2. ngOnChanges (if inputs exist)
3. ngOnInit
4. ngDoCheck
5. ngAfterContentInit
6. ngAfterContentChecked
7. ngAfterViewInit
8. ngAfterViewChecked
```

During subsequent change detection cycles:

```
1. ngOnChanges (if inputs changed)
2. ngDoCheck
3. ngAfterContentChecked
4. ngAfterViewChecked
```

When component is destroyed:

```
1. ngOnDestroy
```

## Detailed Hook Explanations

### ngOnChanges

**Purpose**: Responds when Angular sets or resets data-bound input properties.

**When Called**: 
- Before ngOnInit on initial setup
- Whenever one or more data-bound input properties change

**Use Cases**:
- React to input property changes
- Perform calculations based on multiple inputs
- Validate input values

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  template: `
    <div class="profile">
      <h2>{{ displayName }}</h2>
      <p>Changes detected: {{ changeCount }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnChanges {
  @Input() firstName: string = '';
  @Input() lastName: string = '';
  @Input() age: number = 0;
  
  displayName: string = '';
  changeCount: number = 0;

  ngOnChanges(changes: SimpleChanges): void {
    console.log('ngOnChanges called', changes);
    
    // Check if firstName or lastName changed
    if (changes['firstName'] || changes['lastName']) {
      this.displayName = `${this.firstName} ${this.lastName}`;
      
      // Access previous and current values
      if (changes['firstName']) {
        const firstNameChange = changes['firstName'];
        console.log('First name changed from', 
          firstNameChange.previousValue, 
          'to', 
          firstNameChange.currentValue
        );
      }
    }
    
    // Check if this is the first change
    if (changes['age'] && changes['age'].firstChange) {
      console.log('Age set for the first time:', changes['age'].currentValue);
    }
    
    this.changeCount++;
  }
}
```

### ngOnInit

**Purpose**: Initialize the component after Angular first displays data-bound properties and sets input properties.

**When Called**: Once, after the first ngOnChanges

**Use Cases**:
- Complex initialization logic
- Fetch initial data from services
- Set up subscriptions
- Configure component based on inputs

```typescript
import { Component, Input, OnInit } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-details',
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserDetailsComponent implements OnInit {
  @Input() userId!: string;
  user: any;
  
  constructor(private userService: UserService) {
    // Constructor should only be used for dependency injection
    console.log('Constructor called');
    // Input properties are NOT available here
    console.log('userId in constructor:', this.userId); // undefined
  }

  ngOnInit(): void {
    // ngOnInit is the right place for initialization logic
    console.log('ngOnInit called');
    console.log('userId in ngOnInit:', this.userId); // defined
    
    // Fetch data based on input
    if (this.userId) {
      this.loadUser();
    }
  }
  
  private loadUser(): void {
    this.userService.getUser(this.userId).subscribe(user => {
      this.user = user;
    });
  }
}
```

### ngDoCheck

**Purpose**: Detect and act upon changes that Angular can't or won't detect on its own.

**When Called**: During every change detection run, immediately after ngOnChanges and ngOnInit

**Use Cases**:
- Custom change detection logic
- Detect changes in properties that Angular doesn't track
- Performance optimization (use with caution)

```typescript
import { Component, Input, DoCheck } from '@angular/core';

interface Task {
  id: number;
  title: string;
  completed: boolean;
}

@Component({
  selector: 'app-task-list',
  template: `
    <div>
      <p>Completed: {{ completedCount }} / {{ tasks.length }}</p>
      <p>DoCheck runs: {{ doCheckCount }}</p>
    </div>
  `
})
export class TaskListComponent implements DoCheck {
  @Input() tasks: Task[] = [];
  
  completedCount: number = 0;
  doCheckCount: number = 0;
  private lastTasksString: string = '';

  ngDoCheck(): void {
    this.doCheckCount++;
    
    // Angular doesn't detect changes inside arrays automatically
    // We need to implement our own change detection
    const currentTasksString = JSON.stringify(this.tasks);
    
    if (currentTasksString !== this.lastTasksString) {
      console.log('Tasks array content changed');
      this.completedCount = this.tasks.filter(t => t.completed).length;
      this.lastTasksString = currentTasksString;
    }
  }
}
```

### ngAfterContentInit

**Purpose**: Respond after Angular projects external content into the component's view (ng-content).

**When Called**: Once, after the first ngDoCheck

**Use Cases**:
- Access projected content using @ContentChild/@ContentChildren
- Initialize based on projected content

```typescript
import { Component, ContentChild, AfterContentInit, ElementRef } from '@angular/core';

@Component({
  selector: 'app-card-header',
  template: '<ng-content></ng-content>'
})
export class CardHeaderComponent {}

@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="header">
        <ng-content select="app-card-header"></ng-content>
      </div>
      <div class="body">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class CardComponent implements AfterContentInit {
  @ContentChild(CardHeaderComponent) header?: CardHeaderComponent;

  ngAfterContentInit(): void {
    // Projected content is now available
    console.log('Content projected:', this.header);
    
    if (this.header) {
      console.log('Header element:', this.header);
      // You can now safely access and manipulate projected content
    }
  }
}

// Usage:
// <app-card>
//   <app-card-header>My Card Title</app-card-header>
//   <p>Card content goes here</p>
// </app-card>
```

### ngAfterContentChecked

**Purpose**: Respond after Angular checks the content projected into the component.

**When Called**: After ngAfterContentInit and every subsequent ngDoCheck

**Use Cases**:
- React to changes in projected content
- Rarely needed (use with caution as it runs frequently)

```typescript
import { Component, ContentChildren, AfterContentChecked, QueryList } from '@angular/core';

@Component({
  selector: 'app-tab',
  template: '<ng-content></ng-content>'
})
export class TabComponent {
  @Input() title: string = '';
  @Input() active: boolean = false;
}

@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs">
      <div class="tab-headers">
        <button *ngFor="let tab of tabs" 
                (click)="selectTab(tab)"
                [class.active]="tab.active">
          {{ tab.title }}
        </button>
      </div>
      <div class="tab-content">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class TabsComponent implements AfterContentChecked {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
  
  ngAfterContentChecked(): void {
    // This runs after every change detection cycle
    // Be careful with operations here to avoid performance issues
    console.log('Content checked, tabs count:', this.tabs.length);
  }
  
  selectTab(selectedTab: TabComponent): void {
    this.tabs.forEach(tab => {
      tab.active = (tab === selectedTab);
    });
  }
}
```

### ngAfterViewInit

**Purpose**: Respond after Angular initializes the component's views and child views.

**When Called**: Once, after the first ngAfterContentChecked

**Use Cases**:
- Access child components using @ViewChild/@ViewChildren
- Initialize third-party libraries that need DOM access
- Perform DOM manipulations

```typescript
import { Component, ViewChild, AfterViewInit, ElementRef } from '@angular/core';

@Component({
  selector: 'app-search',
  template: `
    <div>
      <input #searchInput type="text" placeholder="Search...">
      <canvas #chart></canvas>
    </div>
  `
})
export class SearchComponent implements AfterViewInit {
  @ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;
  @ViewChild('chart') chartCanvas!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit(): void {
    // View children are now available
    
    // Auto-focus the input
    this.searchInput.nativeElement.focus();
    
    // Initialize a chart library
    const ctx = this.chartCanvas.nativeElement.getContext('2d');
    if (ctx) {
      // Initialize chart with context
      this.initializeChart(ctx);
    }
    
    console.log('View initialized');
  }
  
  private initializeChart(ctx: CanvasRenderingContext2D): void {
    // Chart initialization logic
  }
}
```

### ngAfterViewChecked

**Purpose**: Respond after Angular checks the component's views and child views.

**When Called**: After ngAfterViewInit and every subsequent ngAfterContentChecked

**Use Cases**:
- React to changes in child component views
- Rarely needed (use with extreme caution as it runs very frequently)

```typescript
import { Component, ViewChild, AfterViewChecked } from '@angular/core';

@Component({
  selector: 'app-child',
  template: '<p>{{ message }}</p>'
})
export class ChildComponent {
  @Input() message: string = '';
}

@Component({
  selector: 'app-parent',
  template: `
    <app-child [message]="parentMessage"></app-child>
    <button (click)="changeMessage()">Change Message</button>
  `
})
export class ParentComponent implements AfterViewChecked {
  @ViewChild(ChildComponent) child!: ChildComponent;
  parentMessage: string = 'Initial message';
  private previousMessage: string = '';

  ngAfterViewChecked(): void {
    // Be very careful here - this runs after EVERY change detection
    if (this.child.message !== this.previousMessage) {
      console.log('Child view updated with message:', this.child.message);
      this.previousMessage = this.child.message;
    }
  }
  
  changeMessage(): void {
    this.parentMessage = 'Updated message';
  }
}
```

### ngOnDestroy

**Purpose**: Cleanup just before Angular destroys the component.

**When Called**: Just before the component is destroyed

**Use Cases**:
- Unsubscribe from observables
- Detach event handlers
- Clear timers and intervals
- Clean up resources to prevent memory leaks

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription, interval } from 'rxjs';
import { DataService } from './data.service';

@Component({
  selector: 'app-timer',
  template: `
    <div>
      <p>Timer: {{ counter }}</p>
      <p>Data updates: {{ dataUpdateCount }}</p>
    </div>
  `
})
export class TimerComponent implements OnInit, OnDestroy {
  counter: number = 0;
  dataUpdateCount: number = 0;
  
  private timerSubscription?: Subscription;
  private dataSubscription?: Subscription;
  private intervalId?: number;

  constructor(private dataService: DataService) {}

  ngOnInit(): void {
    // Observable subscription
    this.timerSubscription = interval(1000).subscribe(() => {
      this.counter++;
    });
    
    // Service subscription
    this.dataSubscription = this.dataService.getData().subscribe(data => {
      this.dataUpdateCount++;
    });
    
    // Native timer
    this.intervalId = window.setInterval(() => {
      console.log('Interval tick');
    }, 2000);
    
    // Event listener
    window.addEventListener('resize', this.onResize);
  }

  ngOnDestroy(): void {
    // Unsubscribe from observables
    if (this.timerSubscription) {
      this.timerSubscription.unsubscribe();
    }
    
    if (this.dataSubscription) {
      this.dataSubscription.unsubscribe();
    }
    
    // Clear native timer
    if (this.intervalId) {
      window.clearInterval(this.intervalId);
    }
    
    // Remove event listeners
    window.removeEventListener('resize', this.onResize);
    
    console.log('Component destroyed, resources cleaned up');
  }
  
  private onResize = (): void => {
    console.log('Window resized');
  }
}
```

## Practical Examples

### Example 1: Lifecycle Logger

```typescript
import { Component, OnInit, OnChanges, DoCheck, AfterContentInit, 
         AfterContentChecked, AfterViewInit, AfterViewChecked, 
         OnDestroy, Input, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-lifecycle-demo',
  template: `
    <div class="lifecycle-demo">
      <h3>{{ title }}</h3>
      <p>Check console for lifecycle events</p>
    </div>
  `
})
export class LifecycleDemoComponent implements OnChanges, OnInit, DoCheck,
    AfterContentInit, AfterContentChecked, AfterViewInit, 
    AfterViewChecked, OnDestroy {
  
  @Input() title: string = '';
  
  private logPrefix = '[LifecycleDemo]';

  constructor() {
    console.log(`${this.logPrefix} Constructor`);
  }

  ngOnChanges(changes: SimpleChanges): void {
    console.log(`${this.logPrefix} ngOnChanges`, changes);
  }

  ngOnInit(): void {
    console.log(`${this.logPrefix} ngOnInit`);
  }

  ngDoCheck(): void {
    console.log(`${this.logPrefix} ngDoCheck`);
  }

  ngAfterContentInit(): void {
    console.log(`${this.logPrefix} ngAfterContentInit`);
  }

  ngAfterContentChecked(): void {
    console.log(`${this.logPrefix} ngAfterContentChecked`);
  }

  ngAfterViewInit(): void {
    console.log(`${this.logPrefix} ngAfterViewInit`);
  }

  ngAfterViewChecked(): void {
    console.log(`${this.logPrefix} ngAfterViewChecked`);
  }

  ngOnDestroy(): void {
    console.log(`${this.logPrefix} ngOnDestroy`);
  }
}
```

### Example 2: Data Fetching Component

```typescript
import { Component, Input, OnInit, OnChanges, OnDestroy, SimpleChanges } from '@angular/core';
import { Subject, takeUntil } from 'rxjs';
import { ApiService } from './api.service';

@Component({
  selector: 'app-data-viewer',
  template: `
    <div class="data-viewer">
      <div *ngIf="loading">Loading...</div>
      <div *ngIf="error" class="error">{{ error }}</div>
      <div *ngIf="data && !loading">
        <pre>{{ data | json }}</pre>
      </div>
    </div>
  `
})
export class DataViewerComponent implements OnInit, OnChanges, OnDestroy {
  @Input() endpoint: string = '';
  @Input() refreshInterval: number = 0;
  
  data: any = null;
  loading: boolean = false;
  error: string = '';
  
  private destroy$ = new Subject<void>();
  private refreshTimer?: number;

  constructor(private apiService: ApiService) {}

  ngOnInit(): void {
    // Initial data load
    this.loadData();
    
    // Set up refresh if interval specified
    if (this.refreshInterval > 0) {
      this.startRefreshTimer();
    }
  }

  ngOnChanges(changes: SimpleChanges): void {
    // Reload data if endpoint changes
    if (changes['endpoint'] && !changes['endpoint'].firstChange) {
      console.log('Endpoint changed, reloading data');
      this.loadData();
    }
    
    // Update refresh timer if interval changes
    if (changes['refreshInterval'] && !changes['refreshInterval'].firstChange) {
      this.stopRefreshTimer();
      if (this.refreshInterval > 0) {
        this.startRefreshTimer();
      }
    }
  }

  ngOnDestroy(): void {
    // Complete the destroy subject to unsubscribe from all observables
    this.destroy$.next();
    this.destroy$.complete();
    
    // Clear refresh timer
    this.stopRefreshTimer();
  }

  private loadData(): void {
    if (!this.endpoint) {
      return;
    }
    
    this.loading = true;
    this.error = '';
    
    this.apiService.get(this.endpoint)
      .pipe(takeUntil(this.destroy$))
      .subscribe({
        next: (data) => {
          this.data = data;
          this.loading = false;
        },
        error: (err) => {
          this.error = err.message || 'Failed to load data';
          this.loading = false;
        }
      });
  }

  private startRefreshTimer(): void {
    this.refreshTimer = window.setInterval(() => {
      this.loadData();
    }, this.refreshInterval);
  }

  private stopRefreshTimer(): void {
    if (this.refreshTimer) {
      window.clearInterval(this.refreshTimer);
      this.refreshTimer = undefined;
    }
  }
}
```

### Example 3: Form with Validation

```typescript
import { Component, Input, OnInit, OnDestroy, AfterViewInit, ViewChild } from '@angular/core';
import { FormGroup, FormControl, Validators } from '@angular/forms';
import { Subject, debounceTime, takeUntil } from 'rxjs';

@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input #firstInput formControlName="username" placeholder="Username">
      <div *ngIf="validationMessages.username" class="error">
        {{ validationMessages.username }}
      </div>
      
      <input formControlName="email" placeholder="Email">
      <div *ngIf="validationMessages.email" class="error">
        {{ validationMessages.email }}
      </div>
      
      <button type="submit" [disabled]="userForm.invalid">Submit</button>
    </form>
  `
})
export class UserFormComponent implements OnInit, AfterViewInit, OnDestroy {
  @Input() initialData?: { username: string; email: string };
  @ViewChild('firstInput') firstInput!: ElementRef<HTMLInputElement>;
  
  userForm!: FormGroup;
  validationMessages: { [key: string]: string } = {};
  
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    // Initialize form
    this.userForm = new FormGroup({
      username: new FormControl(this.initialData?.username || '', [
        Validators.required,
        Validators.minLength(3)
      ]),
      email: new FormControl(this.initialData?.email || '', [
        Validators.required,
        Validators.email
      ])
    });
    
    // Set up validation message updates
    this.userForm.valueChanges
      .pipe(
        debounceTime(300),
        takeUntil(this.destroy$)
      )
      .subscribe(() => {
        this.updateValidationMessages();
      });
  }

  ngAfterViewInit(): void {
    // Focus first input after view initializes
    setTimeout(() => {
      this.firstInput.nativeElement.focus();
    });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private updateValidationMessages(): void {
    this.validationMessages = {};
    
    Object.keys(this.userForm.controls).forEach(key => {
      const control = this.userForm.get(key);
      if (control && control.invalid && control.touched) {
        const errors = control.errors;
        if (errors) {
          if (errors['required']) {
            this.validationMessages[key] = `${key} is required`;
          } else if (errors['minlength']) {
            this.validationMessages[key] = 
              `${key} must be at least ${errors['minlength'].requiredLength} characters`;
          } else if (errors['email']) {
            this.validationMessages[key] = 'Invalid email format';
          }
        }
      }
    });
  }

  onSubmit(): void {
    if (this.userForm.valid) {
      console.log('Form submitted:', this.userForm.value);
    }
  }
}
```

## Best Practices

### 1. Use ngOnInit for Initialization

Always use ngOnInit instead of the constructor for initialization logic:

```typescript
// Bad
constructor(private service: DataService) {
  this.loadData(); // Don't do this
}

// Good
constructor(private service: DataService) {}

ngOnInit(): void {
  this.loadData(); // Do this
}
```

### 2. Clean Up in ngOnDestroy

Always unsubscribe from observables and clean up resources:

```typescript
// Using Subject for automatic unsubscription
export class MyComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.service.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => {
        // Handle data
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Be Cautious with Frequently Called Hooks

Avoid heavy operations in ngDoCheck, ngAfterContentChecked, and ngAfterViewChecked:

```typescript
// Bad
ngAfterViewChecked(): void {
  // This runs on every change detection cycle!
  this.expensiveOperation();
}

// Good
ngAfterViewChecked(): void {
  if (this.needsUpdate) {
    this.expensiveOperation();
    this.needsUpdate = false;
  }
}
```

### 4. Use Lifecycle Hooks for Their Intended Purpose

Each hook has a specific purpose - use the right one:

```typescript
// Access view children in ngAfterViewInit, not ngOnInit
@ViewChild('myElement') element!: ElementRef;

ngOnInit(): void {
  // Don't access view children here - they're undefined
}

ngAfterViewInit(): void {
  // Access view children here
  this.element.nativeElement.focus();
}
```

## Common Mistakes

### 1. Accessing View Children Too Early

```typescript
// Wrong
export class MyComponent implements OnInit {
  @ViewChild('input') input!: ElementRef;

  ngOnInit(): void {
    this.input.nativeElement.focus(); // Error: input is undefined
  }
}

// Correct
export class MyComponent implements AfterViewInit {
  @ViewChild('input') input!: ElementRef;

  ngAfterViewInit(): void {
    this.input.nativeElement.focus(); // Works correctly
  }
}
```

### 2. Forgetting to Unsubscribe

```typescript
// Memory leak
export class MyComponent implements OnInit {
  ngOnInit(): void {
    this.service.getData().subscribe(data => {
      // Subscription never cleaned up
    });
  }
}

// Fixed
export class MyComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.service.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => {
        // Subscription cleaned up properly
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Modifying State in AfterView Hooks

```typescript
// Causes ExpressionChangedAfterItHasBeenCheckedError
export class MyComponent implements AfterViewInit {
  @Input() data: string = '';
  processedData: string = '';

  ngAfterViewInit(): void {
    this.processedData = this.data.toUpperCase(); // Error in dev mode
  }
}

// Fixed
export class MyComponent implements AfterViewInit {
  @Input() data: string = '';
  processedData: string = '';

  ngAfterViewInit(): void {
    setTimeout(() => {
      this.processedData = this.data.toUpperCase(); // Works
    });
  }
}
```

### 4. Heavy Operations in Frequently Called Hooks

```typescript
// Performance problem
export class MyComponent implements DoCheck {
  ngDoCheck(): void {
    // This runs on every change detection!
    this.recalculateEverything();
    this.validateAllData();
  }
}

// Better approach
export class MyComponent implements OnChanges {
  @Input() data: any;

  ngOnChanges(changes: SimpleChanges): void {
    // Only runs when inputs change
    if (changes['data']) {
      this.recalculateEverything();
      this.validateAllData();
    }
  }
}
```

## Interview Questions

### Basic Questions

**Q1: What is the order of lifecycle hooks in Angular?**

A: Constructor → ngOnChanges → ngOnInit → ngDoCheck → ngAfterContentInit → ngAfterContentChecked → ngAfterViewInit → ngAfterViewChecked → ngOnDestroy

**Q2: When should you use ngOnInit vs constructor?**

A: Use constructor only for dependency injection. Use ngOnInit for all initialization logic, data fetching, and accessing input properties.

**Q3: Why is ngOnDestroy important?**

A: ngOnDestroy is crucial for preventing memory leaks by cleaning up subscriptions, event listeners, timers, and other resources before the component is destroyed.

### Intermediate Questions

**Q4: What is the difference between ngAfterContentInit and ngAfterViewInit?**

A: ngAfterContentInit is called after content projection (ng-content) is initialized, while ngAfterViewInit is called after the component's own view and child views are initialized.

**Q5: When would you use ngDoCheck?**

A: Use ngDoCheck for custom change detection when Angular's default change detection doesn't detect changes you need to respond to, such as mutations within arrays or objects.

**Q6: What is SimpleChanges and how is it used?**

A: SimpleChanges is an object passed to ngOnChanges that contains current and previous values of input properties, along with a firstChange boolean indicating if it's the initial change.

### Advanced Questions

**Q7: How do you prevent memory leaks in Angular components?**

A: Use ngOnDestroy to unsubscribe from observables (using takeUntil pattern or Subscription management), clear timers/intervals, remove event listeners, and clean up any resources that could persist after component destruction.

**Q8: What causes ExpressionChangedAfterItHasBeenCheckedError?**

A: This error occurs when you modify component state during change detection in a way that would require another change detection cycle, typically in AfterViewInit or AfterViewChecked hooks. Fix it by using setTimeout or ChangeDetectorRef.detectChanges().

**Q9: How does OnPush change detection strategy affect lifecycle hooks?**

A: With OnPush strategy, change detection (and associated hooks like ngDoCheck, ngAfterContentChecked, ngAfterViewChecked) only runs when input references change, events occur, or observables emit, making the application more performant.

**Q10: Can you modify component state in ngAfterViewInit?**

A: Yes, but if you're updating properties bound to the template, you may need to wrap the update in setTimeout() or trigger change detection manually to avoid ExpressionChangedAfterItHasBeenCheckedError in development mode.

## Resources

### Official Documentation
- [Angular Lifecycle Hooks](https://angular.dev/guide/components/lifecycle)
- [Component Interaction](https://angular.dev/guide/components/inputs)

### Articles and Tutorials
- [Understanding Angular Lifecycle Hooks](https://blog.angular.io/angular-lifecycle-hooks-explained)
- [Mastering Angular Lifecycle Hooks](https://medium.com/angular-in-depth/lifecycle-hooks)

### Video Tutorials
- [Angular Lifecycle Hooks Complete Guide](https://www.youtube.com/watch?v=lifecycle-hooks)
- [Advanced Component Lifecycle Patterns](https://www.youtube.com/watch?v=advanced-lifecycle)

### Books
- "Angular Development with TypeScript" by Yakov Fain and Anton Moiseev
- "ng-book: The Complete Guide to Angular" by Nate Murray

### Tools
- [Angular DevTools](https://angular.dev/tools/devtools) - Debug lifecycle hooks
- [Augury](https://augury.rangle.io/) - Component tree visualization

### Best Practices Guides
- [Angular Style Guide](https://angular.dev/style-guide)
- [Angular Performance Guide](https://angular.dev/best-practices/runtime-performance)
