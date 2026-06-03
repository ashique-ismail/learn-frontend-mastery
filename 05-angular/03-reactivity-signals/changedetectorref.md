# ChangeDetectorRef API

## Table of Contents
- [Introduction](#introduction)
- [What is ChangeDetectorRef](#what-is-changedetectorref)
- [markForCheck](#markforcheck)
- [detectChanges](#detectchanges)
- [detach](#detach)
- [reattach](#reattach)
- [checkNoChanges](#checknochanges)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

ChangeDetectorRef is Angular's API for manually controlling change detection. While Angular's automatic change detection works well most of the time, there are scenarios where manual control is necessary for performance optimization or working with external libraries.

Understanding ChangeDetectorRef is important for:
- Performance optimization with OnPush strategy
- Integration with non-Angular libraries
- Controlling when and how change detection runs
- Debugging change detection issues

## What is ChangeDetectorRef

ChangeDetectorRef provides methods to manipulate the change detection tree:

```typescript
import { Component, ChangeDetectorRef, inject } from '@angular/core';

@Component({
  selector: 'app-cdr-intro',
  template: `
    <div>Value: {{ value }}</div>
    <button (click)="updateValue()">Update</button>
  `
})
export class CdrIntroComponent {
  // Inject ChangeDetectorRef
  private cdr = inject(ChangeDetectorRef);
  
  // Or via constructor
  constructor(private cdrAlt: ChangeDetectorRef) {}
  
  value = 0;
  
  updateValue() {
    // Update value
    this.value++;
    
    // Manually trigger change detection
    this.cdr.detectChanges();
  }
}
```

### Change Detection Hierarchy

Every component has its own ChangeDetectorRef:

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

// Parent Component
@Component({
  selector: 'app-parent',
  template: `
    <div>Parent: {{ parentValue }}</div>
    <app-child></app-child>
  `
})
export class ParentComponent {
  parentValue = 0;
  
  constructor(private parentCdr: ChangeDetectorRef) {
    // This CDR controls parent's change detection
  }
}

// Child Component
@Component({
  selector: 'app-child',
  template: `
    <div>Child: {{ childValue }}</div>
  `
})
export class ChildComponent {
  childValue = 0;
  
  constructor(private childCdr: ChangeDetectorRef) {
    // This CDR controls child's change detection
    // Separate from parent's CDR
  }
}
```

## markForCheck

markForCheck marks the component and all ancestors for check on the next change detection cycle:

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-mark-for-check',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Updated: {{ lastUpdate }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class MarkForCheckComponent {
  counter = 0;
  lastUpdate = new Date();
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Async update outside Angular zone
    setInterval(() => {
      this.counter++;
      this.lastUpdate = new Date();
      
      // Mark this component for checking
      // on the next change detection cycle
      this.cdr.markForCheck();
    }, 1000);
  }
}
```

### markForCheck with Observable

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';
import { interval, takeUntil, Subject } from 'rxjs';

@Component({
  selector: 'app-observable-mark',
  template: `
    <div>Data: {{ data }}</div>
    <div>Status: {{ status }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ObservableMarkComponent {
  data = 0;
  status = 'idle';
  private destroy$ = new Subject<void>();
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Manual subscription requires markForCheck
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(value => {
        this.data = value;
        this.status = value % 2 === 0 ? 'even' : 'odd';
        
        // Required for OnPush to see changes
        this.cdr.markForCheck();
      });
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### markForCheck Propagation

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';

// Grandparent
@Component({
  selector: 'app-grandparent',
  template: `
    <div>Grandparent: {{ grandValue }}</div>
    <app-parent></app-parent>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class GrandparentComponent {
  grandValue = 'unchanged';
}

// Parent
@Component({
  selector: 'app-parent',
  template: `
    <div>Parent: {{ parentValue }}</div>
    <app-child></app-child>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ParentComponent {
  parentValue = 'unchanged';
}

// Child
@Component({
  selector: 'app-child',
  template: `
    <div>Child: {{ childValue }}</div>
    <button (click)="update()">Update</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ChildComponent {
  childValue = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  update() {
    this.childValue++;
    
    // Marks child, parent, and grandparent for check
    // All three will be checked on next CD cycle
    this.cdr.markForCheck();
  }
}
```

### markForCheck vs detectChanges

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-comparison',
  template: `
    <div>Value: {{ value }}</div>
    <button (click)="useMarkForCheck()">markForCheck</button>
    <button (click)="useDetectChanges()">detectChanges</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ComparisonComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  useMarkForCheck() {
    // Schedule check for next CD cycle
    // Waits for Angular's next CD cycle
    setTimeout(() => {
      this.value++;
      this.cdr.markForCheck();
      console.log('Marked, but not checked yet');
    }, 1000);
  }
  
  useDetectChanges() {
    // Immediately check this component and children
    // Runs synchronously
    setTimeout(() => {
      this.value++;
      this.cdr.detectChanges();
      console.log('Already checked');
    }, 1000);
  }
}
```

## detectChanges

detectChanges immediately runs change detection for this component and its children:

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-detect-changes',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Timestamp: {{ timestamp }}</div>
    <app-child [data]="childData"></app-child>
  `
})
export class DetectChangesComponent {
  counter = 0;
  timestamp = Date.now();
  childData = { value: 0 };
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  triggerImmediateUpdate() {
    // Update state
    this.counter++;
    this.timestamp = Date.now();
    this.childData = { value: this.counter };
    
    // Immediately check this component AND children
    this.cdr.detectChanges();
    
    // View is updated synchronously
    console.log('View updated immediately');
  }
}
```

### Synchronous Updates

```typescript
import { Component, ChangeDetectorRef, ViewChild, ElementRef } from '@angular/core';

@Component({
  selector: 'app-sync-update',
  template: `
    <div #display>{{ value }}</div>
    <button (click)="updateAndRead()">Update and Read</button>
  `
})
export class SyncUpdateComponent {
  @ViewChild('display', { static: false }) display!: ElementRef;
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateAndRead() {
    this.value = 42;
    
    // Without detectChanges
    console.log('DOM (before):', this.display.nativeElement.textContent);
    // Logs "0" - DOM not updated yet
    
    // With detectChanges
    this.cdr.detectChanges();
    console.log('DOM (after):', this.display.nativeElement.textContent);
    // Logs "42" - DOM updated immediately
  }
}
```

### detectChanges with Child Components

```typescript
import { Component, ChangeDetectorRef, Input } from '@angular/core';

// Parent
@Component({
  selector: 'app-parent-detect',
  template: `
    <div>Parent: {{ parentValue }}</div>
    <app-child-detect [data]="childValue"></app-child-detect>
    <button (click)="updateAll()">Update All</button>
  `
})
export class ParentDetectComponent {
  parentValue = 0;
  childValue = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateAll() {
    this.parentValue++;
    this.childValue++;
    
    // Checks parent AND child immediately
    this.cdr.detectChanges();
  }
}

// Child
@Component({
  selector: 'app-child-detect',
  template: `
    <div>Child: {{ data }}</div>
    <div>Internal: {{ internal }}</div>
  `
})
export class ChildDetectComponent {
  @Input() data = 0;
  internal = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateInternal() {
    this.internal++;
    
    // Only checks this child, not parent or siblings
    this.cdr.detectChanges();
  }
}
```

## detach

detach removes the component from the change detection tree:

```typescript
import { Component, ChangeDetectorRef, OnInit } from '@angular/core';

@Component({
  selector: 'app-detached',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Status: {{ status }}</div>
    <button (click)="increment()">Increment (won't update)</button>
    <button (click)="manualUpdate()">Manual Update</button>
  `
})
export class DetachedComponent implements OnInit {
  counter = 0;
  status = 'attached';
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Detach from change detection tree
    this.cdr.detach();
    this.status = 'detached';
  }
  
  increment() {
    // Updates model but view won't update
    this.counter++;
    console.log('Counter:', this.counter, 'View: not updated');
  }
  
  manualUpdate() {
    this.counter++;
    
    // Must manually trigger detection
    this.cdr.detectChanges();
  }
}
```

### Performance Optimization with detach

```typescript
import { Component, ChangeDetectorRef, OnInit } from '@angular/core';

@Component({
  selector: 'app-perf-detach',
  template: `
    <div>Updates: {{ updateCount }}</div>
    <div>Last Value: {{ lastValue }}</div>
    <canvas #canvas></canvas>
  `
})
export class PerfDetachComponent implements OnInit {
  updateCount = 0;
  lastValue = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Detach - we'll control CD manually
    this.cdr.detach();
    
    // High-frequency updates
    let count = 0;
    setInterval(() => {
      count++;
      this.lastValue = Math.random();
      
      // Only update view every 100 iterations
      if (count % 100 === 0) {
        this.updateCount = count;
        this.cdr.detectChanges();
      }
    }, 10);
    
    // Without detach: CD runs 100 times/second
    // With detach: CD runs 1 time/second
  }
}
```

### Conditional Change Detection

```typescript
import { Component, ChangeDetectorRef, Input } from '@angular/core';

@Component({
  selector: 'app-conditional-cd',
  template: `
    <div>Mode: {{ mode }}</div>
    <div>Value: {{ value }}</div>
    <div>Updates: {{ updateCount }}</div>
  `
})
export class ConditionalCDComponent {
  @Input() set mode(value: 'auto' | 'manual') {
    this._mode = value;
    
    if (value === 'manual') {
      this.cdr.detach();
    } else {
      this.cdr.reattach();
    }
  }
  
  get mode() {
    return this._mode;
  }
  
  private _mode: 'auto' | 'manual' = 'auto';
  value = 0;
  updateCount = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateValue(newValue: number) {
    this.value = newValue;
    this.updateCount++;
    
    if (this.mode === 'manual') {
      // Must manually detect
      this.cdr.detectChanges();
    }
    // In 'auto' mode, normal CD works
  }
}
```

## reattach

reattach reconnects a detached component to the change detection tree:

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-reattach',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Status: {{ status }}</div>
    <button (click)="toggleDetach()">Toggle Detach</button>
    <button (click)="increment()">Increment</button>
  `
})
export class ReattachComponent {
  counter = 0;
  status = 'attached';
  private isDetached = false;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  toggleDetach() {
    if (this.isDetached) {
      this.cdr.reattach();
      this.status = 'attached';
      this.isDetached = false;
    } else {
      this.cdr.detach();
      this.status = 'detached';
      this.isDetached = true;
    }
  }
  
  increment() {
    this.counter++;
    
    if (this.isDetached) {
      // Need manual detection when detached
      this.cdr.detectChanges();
    }
    // Automatic when attached
  }
}
```

### Dynamic Detach/Reattach Pattern

```typescript
import { Component, ChangeDetectorRef, OnInit, OnDestroy } from '@angular/core';
import { interval, Subject, takeUntil } from 'rxjs';

@Component({
  selector: 'app-dynamic-detach',
  template: `
    <div>Value: {{ value }}</div>
    <div>Mode: {{ isHighFrequency ? 'High Freq' : 'Normal' }}</div>
    <button (click)="toggleMode()">Toggle Mode</button>
  `
})
export class DynamicDetachComponent implements OnInit, OnDestroy {
  value = 0;
  isHighFrequency = false;
  private destroy$ = new Subject<void>();
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Normal frequency updates
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(v => {
        this.value = v;
        
        if (this.isHighFrequency) {
          // Manual detection when detached
          if (v % 10 === 0) {
            this.cdr.detectChanges();
          }
        }
        // Automatic when attached
      });
  }
  
  toggleMode() {
    this.isHighFrequency = !this.isHighFrequency;
    
    if (this.isHighFrequency) {
      // Detach for high-frequency mode
      this.cdr.detach();
    } else {
      // Reattach for normal mode
      this.cdr.reattach();
    }
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## checkNoChanges

checkNoChanges verifies that no changes occurred (development mode only):

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-check-no-changes',
  template: `
    <div>Counter: {{ counter }}</div>
    <button (click)="safeUpdate()">Safe Update</button>
    <button (click)="unsafeUpdate()">Unsafe Update</button>
  `
})
export class CheckNoChangesComponent {
  counter = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  safeUpdate() {
    this.counter++;
    this.cdr.detectChanges();
    
    // Verify no further changes
    // (only in development mode)
    this.cdr.checkNoChanges();
    // No error - state is stable
  }
  
  unsafeUpdate() {
    this.counter++;
    this.cdr.detectChanges();
    
    // Cause additional changes
    setTimeout(() => {
      this.counter++;
    }, 0);
    
    // This will throw in development
    this.cdr.checkNoChanges();
    // Error: Expression has changed after it was checked
  }
}
```

### Debugging with checkNoChanges

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-debug-cd',
  template: `
    <div>Value: {{ getValue() }}</div>
  `
})
export class DebugCDComponent {
  private _value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  getValue() {
    // BAD: Side effects in getter
    this._value++;
    return this._value;
  }
  
  ngAfterViewChecked() {
    // This will catch the above issue
    if (isDevMode()) {
      this.cdr.checkNoChanges();
      // Error: Expression has changed after it was checked
      // getValue() is changing state during CD!
    }
  }
}
```

## Advanced Patterns

### Rate Limiting Change Detection

```typescript
import { Component, ChangeDetectorRef, OnInit, OnDestroy } from '@angular/core';
import { Subject, throttleTime, takeUntil } from 'rxjs';

@Component({
  selector: 'app-rate-limited',
  template: `
    <div>Mouse: {{ x }}, {{ y }}</div>
    <div>Updates: {{ updateCount }}</div>
  `
})
export class RateLimitedComponent implements OnInit, OnDestroy {
  x = 0;
  y = 0;
  updateCount = 0;
  
  private update$ = new Subject<void>();
  private destroy$ = new Subject<void>();
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Detach from automatic CD
    this.cdr.detach();
    
    // Rate limit updates to 60fps max
    this.update$
      .pipe(
        throttleTime(16), // ~60fps
        takeUntil(this.destroy$)
      )
      .subscribe(() => {
        this.cdr.detectChanges();
        this.updateCount++;
      });
    
    // High-frequency mouse tracking
    document.addEventListener('mousemove', this.onMouseMove);
  }
  
  private onMouseMove = (e: MouseEvent) => {
    this.x = e.clientX;
    this.y = e.clientY;
    
    // Request update (rate limited)
    this.update$.next();
  };
  
  ngOnDestroy() {
    document.removeEventListener('mousemove', this.onMouseMove);
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Batch Updates

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-batch-updates',
  template: `
    <div *ngFor="let item of items">
      {{ item.name }}: {{ item.value }}
    </div>
    <button (click)="updateMany()">Update Many</button>
  `
})
export class BatchUpdatesComponent {
  items = Array.from({ length: 100 }, (_, i) => ({
    name: `Item ${i}`,
    value: 0
  }));
  
  constructor(private cdr: ChangeDetectorRef) {
    // Detach for manual control
    this.cdr.detach();
  }
  
  updateMany() {
    // Batch multiple updates
    for (let i = 0; i < this.items.length; i++) {
      this.items[i].value = Math.random();
    }
    
    // Single CD run for all updates
    this.cdr.detectChanges();
    
    // Without detach: 100 CD cycles
    // With detach: 1 CD cycle
  }
}
```

### Lazy Rendering

```typescript
import { Component, ChangeDetectorRef, OnInit } from '@angular/core';

@Component({
  selector: 'app-lazy-render',
  template: `
    <div *ngFor="let item of visibleItems">
      {{ item.name }}
    </div>
  `
})
export class LazyRenderComponent implements OnInit {
  allItems = Array.from({ length: 10000 }, (_, i) => ({
    name: `Item ${i}`
  }));
  
  visibleItems: any[] = [];
  
  constructor(private cdr: ChangeDetectorRef) {
    this.cdr.detach();
  }
  
  ngOnInit() {
    this.loadItemsIncrementally();
  }
  
  private async loadItemsIncrementally() {
    const batchSize = 50;
    
    for (let i = 0; i < this.allItems.length; i += batchSize) {
      // Add batch
      this.visibleItems.push(
        ...this.allItems.slice(i, i + batchSize)
      );
      
      // Render batch
      this.cdr.detectChanges();
      
      // Yield to browser
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}
```

### Integration with External Libraries

```typescript
import { Component, ChangeDetectorRef, ElementRef, ViewChild } from '@angular/core';

// Example: Integrating D3.js
declare const d3: any;

@Component({
  selector: 'app-d3-integration',
  template: `
    <div>Selected: {{ selected }}</div>
    <svg #chart></svg>
  `
})
export class D3IntegrationComponent {
  @ViewChild('chart', { static: true }) chartRef!: ElementRef;
  selected: string | null = null;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    const svg = d3.select(this.chartRef.nativeElement);
    
    // D3 updates happen outside Angular
    svg.selectAll('circle')
      .data([1, 2, 3])
      .enter()
      .append('circle')
      .on('click', (event: any, d: number) => {
        // D3 event handler
        this.selected = `Circle ${d}`;
        
        // Must manually trigger CD
        this.cdr.detectChanges();
      });
  }
}
```

## Common Mistakes

### Mistake 1: Using detectChanges in OnPush Without Need

```typescript
// BAD: Unnecessary detectChanges
@Component({
  selector: 'app-unnecessary',
  template: `<div>{{ value() }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UnnecessaryComponent {
  value = signal(0);
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  update() {
    this.value.set(42);
    this.cdr.detectChanges(); // NOT NEEDED - signals handle it
  }
}

// GOOD: Let signals handle it
@Component({
  selector: 'app-good',
  template: `<div>{{ value() }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class GoodComponent {
  value = signal(0);
  
  update() {
    this.value.set(42); // Automatic CD
  }
}
```

### Mistake 2: Forgetting to Reattach

```typescript
// BAD: Detached forever
@Component({
  selector: 'app-forgot-reattach',
  template: `<div>{{ value }}</div>`
})
export class ForgotReattachComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {
    this.cdr.detach();
    
    // Later updates won't work!
    setInterval(() => {
      this.value++;
      // Forgot to call detectChanges or reattach
    }, 1000);
  }
}

// GOOD: Reattach when done
@Component({
  selector: 'app-remember-reattach',
  template: `<div>{{ value }}</div>`
})
export class RememberReattachComponent implements OnDestroy {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {
    this.cdr.detach();
    
    // Do work...
  }
  
  ngOnDestroy() {
    // Reattach before destroying
    this.cdr.reattach();
  }
}
```

### Mistake 3: Overusing Manual CD

```typescript
// BAD: Manual CD everywhere
@Component({
  selector: 'app-overuse',
  template: `<div>{{ user | json }}</div>`
})
export class OveruseComponent {
  user = { name: '', age: 0 };
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateUser() {
    this.user.name = 'John';
    this.cdr.detectChanges(); // Not needed with default CD
    
    this.user.age = 30;
    this.cdr.detectChanges(); // Still not needed!
  }
}

// GOOD: Use signals or OnPush appropriately
@Component({
  selector: 'app-appropriate',
  template: `<div>{{ user() | json }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AppropriateComponent {
  user = signal({ name: '', age: 0 });
  
  updateUser() {
    this.user.set({ name: 'John', age: 30 });
    // Automatic CD with OnPush + signals
  }
}
```

## Best Practices

### 1. Prefer Signals Over Manual CD

```typescript
// Instead of manual CD, use signals
@Component({
  selector: 'app-modern',
  template: `<div>{{ state() | json }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ModernComponent {
  state = signal({ count: 0 });
  
  async fetchData() {
    const data = await fetch('/api/data').then(r => r.json());
    this.state.set(data); // Automatic CD
  }
}
```

### 2. Use markForCheck for Async Operations

```typescript
// For async operations with OnPush
@Component({
  selector: 'app-async-ops',
  template: `<div>{{ data }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AsyncOpsComponent {
  data = '';
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  async loadData() {
    const result = await fetch('/api/data').then(r => r.json());
    this.data = result;
    this.cdr.markForCheck(); // Schedule check
  }
}
```

### 3. Detach Only When Necessary

```typescript
// Detach for high-frequency updates
@Component({
  selector: 'app-high-freq',
  template: `<canvas #canvas></canvas>`
})
export class HighFreqComponent {
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Detach for animation loop
    this.cdr.detach();
    
    const animate = () => {
      // Update canvas
      // No CD needed for canvas updates
      requestAnimationFrame(animate);
    };
    animate();
  }
}
```

### 4. Always Pair detach with reattach

```typescript
@Component({
  selector: 'app-paired',
  template: `<div>{{ value }}</div>`
})
export class PairedComponent implements OnDestroy {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {
    this.cdr.detach();
    // Do optimized work...
  }
  
  ngOnDestroy() {
    // Always reattach in cleanup
    this.cdr.reattach();
  }
}
```

## Interview Questions

### Q1: What's the difference between markForCheck and detectChanges?
**Answer:** markForCheck schedules the component and its ancestors for checking on the next change detection cycle (asynchronous). detectChanges immediately runs change detection for the component and its children (synchronous). Use markForCheck with OnPush for async updates, and detectChanges when you need immediate synchronous updates.

### Q2: When should you use detach?
**Answer:** Use detach when you have high-frequency updates and want complete control over when change detection runs. Common use cases include: canvas/WebGL animations, mouse tracking, real-time data streams, or third-party library integrations. Always remember to either call detectChanges manually or reattach when done.

### Q3: Why is markForCheck needed with OnPush?
**Answer:** OnPush only checks a component when input references change or events fire from its template. For async operations like setTimeout, HTTP responses, or manual subscriptions, these triggers don't occur. markForCheck tells Angular "please check this component on the next cycle" even though the normal OnPush conditions weren't met.

### Q4: Does detectChanges check parent components?
**Answer:** No, detectChanges only checks the current component and its children (downward in the tree). It does not check ancestors. markForCheck, however, marks the component and all ancestors upward to the root.

### Q5: What happens if you detach a component and forget to reattach?
**Answer:** The component will be permanently detached from change detection. View updates will never happen automatically, even for events and input changes. You must manually call detectChanges for any updates, or reattach to restore normal change detection behavior.

## Key Takeaways

1. ChangeDetectorRef provides manual control over change detection
2. markForCheck schedules a check (async), detectChanges runs immediately (sync)
3. markForCheck is essential for OnPush with async operations
4. detach removes a component from the CD tree for optimization
5. Always pair detach with reattach in cleanup
6. Prefer signals over manual change detection when possible
7. detectChanges checks component and children, markForCheck marks component and ancestors
8. Use detach for high-frequency updates (animations, tracking)
9. checkNoChanges is a development-only debugging tool
10. Manual CD is an optimization technique - don't overuse it

## Resources

### Official Documentation
- [ChangeDetectorRef API](https://angular.dev/api/core/ChangeDetectorRef)
- [Change Detection Guide](https://angular.dev/best-practices/runtime-performance)
- [OnPush Strategy](https://angular.dev/api/core/ChangeDetectionStrategy)

### Articles
- [Understanding ChangeDetectorRef](https://blog.angular.io/change-detection-explained)
- [Performance Optimization with detach](https://angular.dev/guide/change-detection-optimization)
- [Manual Change Detection Patterns](https://blog.angular.io/advanced-cd-patterns)

### Tools
- Angular DevTools
- Chrome Performance profiler
- Change detection visualizer

### Videos
- "Change Detection Deep Dive" - Angular team
- "Performance Optimization" talks
