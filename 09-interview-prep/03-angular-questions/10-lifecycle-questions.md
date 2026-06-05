# Angular Lifecycle Hooks Interview Questions

## The Idea

**In plain English:** Lifecycle hooks are special functions that Angular automatically calls at specific moments during a component's life — like when it first appears on screen, when it receives new data, or when it disappears. Think of them as checkpoints where you can run your own code at exactly the right time.

**Real-world analogy:** Think of opening and running a pop-up food stall at a market. Before you serve customers you set up your equipment, during the day you restock and adjust, and at the end of the day you pack everything away and clean up so you leave no mess behind.

- The stall setup phase = `ngOnInit` (prepare everything before the component is visible)
- Restocking and adjusting during the day = `ngOnChanges` / `ngDoCheck` (react to new data or check for updates)
- Packing up and cleaning at the end = `ngOnDestroy` (cancel subscriptions and timers so nothing leaks)

---

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

Angular lifecycle hooks are methods that give visibility into key moments in a component's life, from creation to destruction. Understanding these hooks is essential for proper initialization, cleanup, and optimization.

### Lifecycle Sequence

1. **ngOnChanges** - When input properties change
2. **ngOnInit** - After first ngOnChanges
3. **ngDoCheck** - During every change detection
4. **ngAfterContentInit** - After content projection
5. **ngAfterContentChecked** - After content checked
6. **ngAfterViewInit** - After view initialization
7. **ngAfterViewChecked** - After view checked
8. **ngOnDestroy** - Before component destruction

## Common Interview Questions

### Q1: Explain the difference between ngOnInit and constructor. When should you use each?

**What the interviewer wants to know:**
- Understanding of initialization timing
- Knowledge of Angular lifecycle
- Proper initialization patterns

**Strong Answer:**

The constructor and ngOnInit serve different purposes in component initialization. Understanding when to use each is fundamental to Angular development.

**Constructor - TypeScript/JavaScript Feature:**

```typescript
import { Component } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-profile'
})
export class UserProfileComponent {
  // Constructor runs before Angular initializes the component
  // At this point, inputs are NOT available
  // View is NOT initialized
  
  constructor(
    private userService: UserService,
    private http: HttpClient
  ) {
    // ✅ GOOD: Dependency injection
    console.log('Constructor called');
    
    // ✅ GOOD: Simple property initialization
    this.loading = false;
    this.error = null;
    
    // ❌ BAD: Accessing @Input() properties
    // console.log(this.userId); // undefined!
    
    // ❌ BAD: Accessing view children
    // console.log(this.nameInput); // undefined!
    
    // ❌ BAD: Complex initialization logic
    // this.loadData(); // Should be in ngOnInit
    
    // ❌ BAD: API calls
    // this.http.get('/api/data').subscribe(); // Should be in ngOnInit
  }
}
```

**ngOnInit - Angular Lifecycle Hook:**

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId: number;
  user: User;
  
  constructor(private userService: UserService) {
    // Constructor runs first
    console.log('Constructor:', this.userId); // undefined
  }
  
  ngOnInit(): void {
    // ngOnInit runs after constructor
    // At this point, @Input() properties are available
    console.log('ngOnInit:', this.userId); // value available
    
    // ✅ GOOD: Access input properties
    if (this.userId) {
      this.loadUser();
    }
    
    // ✅ GOOD: Initialize component data
    this.initializeForm();
    
    // ✅ GOOD: API calls
    this.loadData();
    
    // ✅ GOOD: Subscribe to observables
    this.setupSubscriptions();
    
    // ✅ GOOD: Set up timers/intervals
    this.startPolling();
  }
  
  private loadUser(): void {
    this.userService.getUser(this.userId).subscribe(user => {
      this.user = user;
    });
  }
}
```

**Execution Order Example:**

```typescript
@Component({
  selector: 'app-demo'
})
export class DemoComponent implements OnInit, AfterViewInit {
  @Input() message: string;
  @ViewChild('input') inputElement: ElementRef;
  
  constructor() {
    console.log('1. Constructor');
    console.log('   @Input message:', this.message); // undefined
    console.log('   @ViewChild input:', this.inputElement); // undefined
  }
  
  ngOnChanges(changes: SimpleChanges): void {
    console.log('2. ngOnChanges');
    console.log('   @Input message:', this.message); // available
    console.log('   @ViewChild input:', this.inputElement); // still undefined
  }
  
  ngOnInit(): void {
    console.log('3. ngOnInit');
    console.log('   @Input message:', this.message); // available
    console.log('   @ViewChild input:', this.inputElement); // still undefined
  }
  
  ngAfterViewInit(): void {
    console.log('4. ngAfterViewInit');
    console.log('   @Input message:', this.message); // available
    console.log('   @ViewChild input:', this.inputElement); // NOW available
  }
}
```

**Follow-up Questions:**

1. **Can you call ngOnInit multiple times?**
   - No, Angular calls ngOnInit once per component instance
   - If you need to re-initialize, extract logic into separate method
   - Use ngOnChanges to react to input changes

2. **What happens if you don't implement ngOnInit?**
   - Nothing - it's optional
   - Component will still work
   - But you miss proper initialization hook

3. **Should services be initialized in constructor or ngOnInit?**
   - Inject in constructor
   - Call methods in ngOnInit
   - This follows dependency injection pattern

---

### Q2: Explain ngOnDestroy and proper cleanup patterns.

**What the interviewer wants to know:**
- Understanding of component lifecycle
- Memory leak prevention
- Resource management

**Strong Answer:**

ngOnDestroy is called immediately before Angular destroys a component. It's critical for cleaning up resources to prevent memory leaks.

**Basic ngOnDestroy:**

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-cleanup'
})
export class CleanupComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // Set up resources
    this.setupSubscriptions();
    this.startPolling();
  }
  
  ngOnDestroy(): void {
    // Clean up resources
    console.log('Component destroying');
    
    // Trigger takeUntil for all subscriptions
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  private setupSubscriptions(): void {
    this.dataService.data$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.data = data;
    });
  }
}
```

**Common Resources to Clean Up:**

```typescript
@Component({})
export class ResourceCleanupComponent implements OnDestroy {
  // 1. Observable subscriptions
  private subscription: Subscription;
  private destroy$ = new Subject<void>();
  
  // 2. Timers/Intervals
  private intervalId: any;
  private timeoutId: any;
  
  // 3. Event listeners
  private clickHandler: () => void;
  
  // 4. WebSocket connections
  private ws: WebSocket;
  
  // 5. Third-party libraries
  private chart: any;
  
  ngOnInit(): void {
    // Set up resources
    
    // 1. Subscriptions
    this.subscription = this.dataService.getData().subscribe();
    
    // 2. Timers
    this.intervalId = setInterval(() => this.poll(), 5000);
    this.timeoutId = setTimeout(() => this.doSomething(), 10000);
    
    // 3. Event listeners
    this.clickHandler = () => this.handleClick();
    document.addEventListener('click', this.clickHandler);
    
    // 4. WebSocket
    this.ws = new WebSocket('ws://localhost:8080');
    this.ws.onmessage = (event) => this.handleMessage(event);
    
    // 5. Third-party library
    this.chart = createChart('#chart-container');
  }
  
  ngOnDestroy(): void {
    // Clean up ALL resources
    
    // 1. Unsubscribe
    this.subscription?.unsubscribe();
    this.destroy$.next();
    this.destroy$.complete();
    
    // 2. Clear timers
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }
    
    // 3. Remove event listeners
    if (this.clickHandler) {
      document.removeEventListener('click', this.clickHandler);
    }
    
    // 4. Close WebSocket
    if (this.ws) {
      this.ws.close();
    }
    
    // 5. Destroy third-party resources
    if (this.chart) {
      this.chart.destroy();
    }
  }
}
```

**takeUntil Pattern (Recommended):**

```typescript
@Component({})
export class TakeUntilPatternComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // All subscriptions use takeUntil
    this.dataService.data$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => this.data = data);
    
    this.userService.user$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(user => this.user = user);
    
    interval(5000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => this.poll());
  }
  
  ngOnDestroy(): void {
    // Single cleanup triggers all takeUntil
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Follow-up Questions:**

1. **When exactly is ngOnDestroy called?**
   - Before Angular removes component from DOM
   - When parent component destroys
   - When navigating away from route
   - Before app closes

2. **What happens if you don't implement ngOnDestroy?**
   - Memory leaks from unclosed subscriptions
   - Event listeners continue firing
   - Timers keep running
   - Can cause performance issues

3. **Can you prevent component destruction?**
   - No, ngOnDestroy is informational only
   - Use canDeactivate guard to prevent navigation
   - ngOnDestroy always runs when component destroys

---

## What Interviewers Look For

### Strong Signals

1. **Lifecycle Understanding**: Clear knowledge of hook order and purpose
2. **Proper Initialization**: Uses ngOnInit correctly, not constructor
3. **Cleanup Discipline**: Implements proper ngOnDestroy patterns
4. **Memory Leak Awareness**: Knows common leak sources
5. **Best Practices**: Uses takeUntil pattern, async pipe
6. **Real Experience**: Shares debugging stories

## Red Flags to Avoid

1. **"Constructor and ngOnInit are the same"**: Fundamental misunderstanding
2. **No cleanup in ngOnDestroy**: Memory leak issues
3. **Can't explain lifecycle order**: Basic knowledge gap
4. **Accesses ViewChild in ngOnInit**: Timing misunderstanding
5. **No takeUntil pattern knowledge**: Outdated practices
6. **"Never had memory leaks"**: Lack of awareness or experience
7. **Can't debug lifecycle issues**: Limited troubleshooting skills

## Key Takeaways

### Essential Concepts

1. **Constructor vs ngOnInit**
   - Constructor: DI, simple initialization
   - ngOnInit: Complex initialization, access inputs

2. **Lifecycle Order**
   - ngOnChanges → ngOnInit → ngDoCheck → ngAfterContentInit → ngAfterContentChecked → ngAfterViewInit → ngAfterViewChecked → ngOnDestroy

3. **ngOnDestroy**
   - Critical for cleanup
   - Prevents memory leaks
   - Use takeUntil pattern

### Best Practices

1. Inject dependencies in constructor
2. Initialize in ngOnInit
3. Clean up in ngOnDestroy
4. Use takeUntil for subscriptions
5. Access ViewChild in ngAfterViewInit
6. Use async pipe when possible
7. Implement proper resource management

### Interview Success Tips

1. **Start with basics**: Explain constructor vs ngOnInit clearly
2. **Show lifecycle knowledge**: Understand complete sequence
3. **Emphasize cleanup**: Always mention ngOnDestroy
4. **Share real examples**: Memory leak debugging stories
5. **Discuss patterns**: takeUntil, async pipe
6. **Ask clarifying questions**: Understand the scenario
7. **Be honest**: Admit knowledge gaps

### Quick Reference

```typescript
// Lifecycle hooks order
constructor() → ngOnChanges() → ngOnInit() → ngDoCheck() → 
ngAfterContentInit() → ngAfterContentChecked() → 
ngAfterViewInit() → ngAfterViewChecked() → ngOnDestroy()

// Constructor: DI only
constructor(private service: Service) {}

// ngOnInit: Initialization
ngOnInit(): void {
  this.loadData();
}

// ngOnDestroy: Cleanup
ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}

// takeUntil pattern
private destroy$ = new Subject<void>();
this.data$.pipe(takeUntil(this.destroy$)).subscribe();
```

Remember: Lifecycle hooks are about timing and proper resource management. Master initialization and cleanup to build robust, leak-free Angular applications.
