# OnPush Change Detection Strategy

## The Idea

**In plain English:** OnPush is a setting you apply to a part of your app that tells Angular "only re-check this section when something relevant to it actually changes" — instead of constantly re-checking everything every single time anything happens anywhere on the page. Angular's "change detection" is just its way of comparing what the screen shows to what the data says and updating the screen if they differ.

**Real-world analogy:** Imagine a hotel with 100 rooms. The default manager checks every single room after every guest phone call to see if anything needs cleaning. The OnPush manager is smarter: they only check a room if a guest in that specific room pressed the call button, a new guest just checked in, or the automated system sent an alert for that room.

- The hotel = the Angular application
- Each room = a component on the page
- The manager checking a room = Angular running change detection on a component
- A guest pressing the call button = an event or input change originating from that component

---

## Table of Contents

- [Introduction](#introduction)
- [Default vs OnPush](#default-vs-onpush)
- [How OnPush Works](#how-onpush-works)
- [When OnPush Triggers](#when-onpush-triggers)
- [OnPush with Signals](#onpush-with-signals)
- [OnPush with Observables](#onpush-with-observables)
- [Performance Benefits](#performance-benefits)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

OnPush is an optimization strategy that tells Angular to only check a component for changes when specific conditions are met, rather than on every change detection cycle. This dramatically improves performance in large applications.

Understanding OnPush is critical for:
- Building performant Angular applications
- Optimizing change detection cycles
- Working effectively with immutable data patterns
- Preparing for zoneless Angular

## Default vs OnPush

### Default Change Detection

```typescript
import { Component } from '@angular/core';

// Default strategy checks EVERY component on EVERY event
@Component({
  selector: 'app-default',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Heavy computation: {{ expensiveOperation() }}</div>
  `
  // changeDetection: ChangeDetectionStrategy.Default (implicit)
})
export class DefaultComponent {
  counter = 0;
  
  expensiveOperation() {
    console.log('Running expensive operation');
    let result = 0;
    for (let i = 0; i < 1000000; i++) {
      result += Math.random();
    }
    return result;
  }
  
  // Even if counter doesn't change, expensiveOperation runs
  // on EVERY change detection cycle in the entire app!
}
```

### OnPush Change Detection

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-onpush',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Heavy computation: {{ expensiveOperation() }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnPushComponent {
  counter = 0;
  
  expensiveOperation() {
    console.log('Running expensive operation');
    let result = 0;
    for (let i = 0; i < 1000000; i++) {
      result += Math.random();
    }
    return result;
  }
  
  // Only runs when OnPush conditions are met
  // Dramatically reduces unnecessary computation
}
```

### Performance Comparison

```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';

// Parent with many children
@Component({
  selector: 'app-parent',
  template: `
    <div>
      <app-child-default 
        *ngFor="let item of items" 
        [data]="item">
      </app-child-default>
    </div>
    <button (click)="unrelatedAction()">Unrelated Action</button>
  `
})
export class ParentComponent {
  items = Array.from({ length: 100 }, (_, i) => ({ id: i }));
  
  unrelatedAction() {
    console.log('Action happened');
    // With Default: ALL 100 children check for changes
    // With OnPush: NO children check (inputs didn't change)
  }
}

// Default child - checks on every parent event
@Component({
  selector: 'app-child-default',
  template: `<div>{{ data.id }}</div>`
})
export class ChildDefaultComponent {
  @Input() data!: any;
  
  ngDoCheck() {
    console.log('Checking child', this.data.id);
    // Logs 100 times on every parent action!
  }
}

// OnPush child - only checks when input reference changes
@Component({
  selector: 'app-child-onpush',
  template: `<div>{{ data.id }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ChildOnPushComponent {
  @Input() data!: any;
  
  ngDoCheck() {
    console.log('Checking child', this.data.id);
    // Only logs when data reference changes
  }
}
```

## How OnPush Works

### Reference Checking

OnPush relies on reference equality, not deep equality:

```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';

@Component({
  selector: 'app-reference-demo',
  template: `
    <div>Name: {{ user.name }}</div>
    <div>Age: {{ user.age }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ReferenceDemoComponent {
  @Input() user!: { name: string; age: number };
  
  ngOnChanges() {
    console.log('OnChanges triggered');
  }
}

// Parent component
@Component({
  selector: 'app-parent',
  template: `
    <app-reference-demo [user]="user"></app-reference-demo>
    <button (click)="mutateUser()">Mutate User (BAD)</button>
    <button (click)="replaceUser()">Replace User (GOOD)</button>
  `
})
export class ParentComponent {
  user = { name: 'John', age: 30 };
  
  mutateUser() {
    // BAD: Mutates existing object
    this.user.age = 31;
    // OnPush child won't detect this change!
    // Reference is the same
  }
  
  replaceUser() {
    // GOOD: Creates new reference
    this.user = { ...this.user, age: 31 };
    // OnPush child WILL detect this change
    // New reference triggers change detection
  }
}
```

### Change Detection Tree

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';

// Grandparent (Default)
@Component({
  selector: 'app-grandparent',
  template: `
    <app-parent [data]="data"></app-parent>
    <button (click)="onClick()">Click</button>
  `
})
export class GrandparentComponent {
  data = { value: 0 };
  
  onClick() {
    console.log('Grandparent clicked');
    // With Default: checks grandparent, parent, child
    // With OnPush on parent & child: only checks grandparent
  }
}

// Parent (OnPush)
@Component({
  selector: 'app-parent',
  template: `
    <app-child [data]="data"></app-child>
    <div>Parent: {{ localValue }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ParentComponent {
  @Input() data!: any;
  localValue = 0;
  
  // Parent only checks when:
  // 1. Input reference changes
  // 2. Event originates from this component
  // 3. Manually triggered
  // 4. Async pipe emits
}

// Child (OnPush)
@Component({
  selector: 'app-child',
  template: `<div>Child: {{ data.value }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ChildComponent {
  @Input() data!: any;
}
```

## When OnPush Triggers

### 1. Input Reference Change

```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';

@Component({
  selector: 'app-input-trigger',
  template: `
    <div>Data: {{ data | json }}</div>
    <div>Render count: {{ renderCount() }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class InputTriggerComponent {
  @Input() data: any;
  private count = 0;
  
  renderCount() {
    return ++this.count;
  }
}

// Parent
@Component({
  selector: 'app-parent',
  template: `
    <app-input-trigger [data]="user"></app-input-trigger>
    <button (click)="updateWrong()">Update Wrong</button>
    <button (click)="updateRight()">Update Right</button>
  `
})
export class ParentComponent {
  user = { name: 'John', age: 30 };
  
  updateWrong() {
    // Doesn't trigger OnPush
    this.user.age = 31;
  }
  
  updateRight() {
    // Triggers OnPush - new reference
    this.user = { ...this.user, age: 31 };
  }
}
```

### 2. Component Events

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-event-trigger',
  template: `
    <div>Counter: {{ counter }}</div>
    <button (click)="increment()">Increment</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class EventTriggerComponent {
  counter = 0;
  
  increment() {
    // Event handlers trigger change detection
    // for the component AND its ancestors
    this.counter++;
  }
}
```

### 3. Async Pipe

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { interval, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-async-trigger',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <div>Time: {{ time$ | async }}</div>
    <div>Counter: {{ counter$ | async }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AsyncTriggerComponent {
  // Async pipe automatically triggers change detection
  time$: Observable<Date> = interval(1000).pipe(
    map(() => new Date())
  );
  
  counter$ = interval(1000);
  
  // No manual change detection needed!
  // Async pipe calls markForCheck internally
}
```

### 4. Manual Trigger

```typescript
import { 
  Component, 
  ChangeDetectionStrategy, 
  ChangeDetectorRef 
} from '@angular/core';

@Component({
  selector: 'app-manual-trigger',
  template: `
    <div>Value: {{ value }}</div>
    <button (click)="updateWithoutCD()">Update Without CD</button>
    <button (click)="updateWithCD()">Update With CD</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ManualTriggerComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateWithoutCD() {
    // Async callback - not an event from template
    setTimeout(() => {
      this.value++;
      // View won't update - OnPush not triggered
    }, 1000);
  }
  
  updateWithCD() {
    setTimeout(() => {
      this.value++;
      // Manually trigger change detection
      this.cdr.markForCheck();
    }, 1000);
  }
}
```

## OnPush with Signals

Signals work perfectly with OnPush:

```typescript
import { Component, ChangeDetectionStrategy, signal, computed } from '@angular/core';

@Component({
  selector: 'app-signals-onpush',
  template: `
    <div>Count: {{ count() }}</div>
    <div>Double: {{ double() }}</div>
    <div>Message: {{ message() }}</div>
    <button (click)="increment()">Increment</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SignalsOnPushComponent {
  // Signals automatically trigger OnPush
  count = signal(0);
  double = computed(() => this.count() * 2);
  message = signal('Hello');
  
  increment() {
    // Signal update automatically marks for check
    this.count.update(c => c + 1);
  }
  
  async fetchData() {
    const data = await fetch('/api/data').then(r => r.json());
    // Automatically triggers change detection
    this.message.set(data.message);
  }
}
```

### Signal Inputs with OnPush

```typescript
import { Component, ChangeDetectionStrategy, input, computed } from '@angular/core';

@Component({
  selector: 'app-signal-input',
  template: `
    <div>Name: {{ name() }}</div>
    <div>Upper: {{ upperName() }}</div>
    <div>Age: {{ age() }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SignalInputComponent {
  // Signal inputs automatically work with OnPush
  name = input.required<string>();
  age = input<number>(0);
  
  // Computed values update automatically
  upperName = computed(() => this.name().toUpperCase());
}

// Parent
@Component({
  selector: 'app-parent',
  template: `
    <app-signal-input 
      [name]="userName()" 
      [age]="userAge()">
    </app-signal-input>
    <button (click)="updateUser()">Update</button>
  `
})
export class ParentComponent {
  userName = signal('John');
  userAge = signal(30);
  
  updateUser() {
    // Automatically triggers child OnPush
    this.userName.set('Jane');
    this.userAge.set(25);
  }
}
```

### Effects with OnPush

```typescript
import { Component, ChangeDetectionStrategy, signal, effect } from '@angular/core';

@Component({
  selector: 'app-effects-onpush',
  template: `
    <div>Count: {{ count() }}</div>
    <div>Log: {{ log() }}</div>
    <button (click)="increment()">Increment</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class EffectsOnPushComponent {
  count = signal(0);
  log = signal('');
  
  constructor() {
    // Effects run when dependencies change
    effect(() => {
      const current = this.count();
      console.log('Count changed:', current);
      
      // Updating another signal in effect
      this.log.set(`Last count: ${current}`);
      // This triggers OnPush automatically
    });
  }
  
  increment() {
    this.count.update(c => c + 1);
    // Effect runs, updates log signal, view updates
  }
}
```

## OnPush with Observables

### Basic Observable Usage

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';
import { interval } from 'rxjs';

@Component({
  selector: 'app-observable-onpush',
  template: `
    <div>Counter: {{ counter }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ObservableOnPushComponent {
  counter = 0;
  
  constructor(private cdr: ChangeDetectorRef) {
    // Manual subscription requires manual CD
    interval(1000).subscribe(value => {
      this.counter = value;
      this.cdr.markForCheck(); // Required for OnPush
    });
  }
}
```

### Async Pipe (Recommended)

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { interval, map, Observable } from 'rxjs';

@Component({
  selector: 'app-async-pipe-onpush',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <div>Counter: {{ counter$ | async }}</div>
    <div>Doubled: {{ doubled$ | async }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AsyncPipeOnPushComponent {
  counter$ = interval(1000);
  
  doubled$ = this.counter$.pipe(
    map(value => value * 2)
  );
  
  // Async pipe handles markForCheck automatically!
}
```

### toSignal Conversion

```typescript
import { Component, ChangeDetectionStrategy, computed } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { interval, map } from 'rxjs';

@Component({
  selector: 'app-tosignal-onpush',
  template: `
    <div>Counter: {{ counter() }}</div>
    <div>Doubled: {{ doubled() }}</div>
    <div>Status: {{ status() }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ToSignalOnPushComponent {
  private counter$ = interval(1000);
  
  // Convert observable to signal
  counter = toSignal(this.counter$, { initialValue: 0 });
  
  // Computed signals work seamlessly
  doubled = computed(() => this.counter() * 2);
  
  status = computed(() => {
    const count = this.counter();
    return count < 10 ? 'Starting' : 'Running';
  });
  
  // All updates automatically trigger OnPush!
}
```

## Performance Benefits

### Measurement Example

```typescript
import { Component, ChangeDetectionStrategy, Input, OnInit } from '@angular/core';

@Component({
  selector: 'app-perf-measurement',
  template: `
    <div>
      <h3>Performance Test</h3>
      <app-child-list [items]="items"></app-child-list>
      <button (click)="unrelatedAction()">Unrelated Action</button>
    </div>
  `
})
export class PerfMeasurementComponent implements OnInit {
  items: any[] = [];
  
  ngOnInit() {
    // Create 1000 items
    this.items = Array.from({ length: 1000 }, (_, i) => ({
      id: i,
      name: `Item ${i}`
    }));
  }
  
  unrelatedAction() {
    console.time('change-detection');
    console.log('Unrelated action');
    console.timeEnd('change-detection');
  }
}

// Child list with Default
@Component({
  selector: 'app-child-list-default',
  template: `
    <div *ngFor="let item of items">
      <app-item-default [item]="item"></app-item-default>
    </div>
  `
})
export class ChildListDefaultComponent {
  @Input() items: any[] = [];
}

// Child list with OnPush
@Component({
  selector: 'app-child-list-onpush',
  template: `
    <div *ngFor="let item of items">
      <app-item-onpush [item]="item"></app-item-onpush>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ChildListOnPushComponent {
  @Input() items: any[] = [];
}

// Individual item - Default (checks every time)
@Component({
  selector: 'app-item-default',
  template: `<div>{{ item.name }}</div>`
})
export class ItemDefaultComponent {
  @Input() item: any;
  
  ngDoCheck() {
    // Called 1000 times on every parent action!
  }
}

// Individual item - OnPush (checks only when input changes)
@Component({
  selector: 'app-item-onpush',
  template: `<div>{{ item.name }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ItemOnPushComponent {
  @Input() item: any;
  
  ngDoCheck() {
    // Only called when item reference changes
  }
}
```

### Real-world Performance Impact

```typescript
import { Component, ChangeDetectionStrategy, signal } from '@angular/core';

@Component({
  selector: 'app-performance-demo',
  template: `
    <div>
      <h3>Large List Demo</h3>
      <div>Items: {{ items().length }}</div>
      <div>Selected: {{ selectedCount() }}</div>
      <button (click)="generateItems()">Generate 10k Items</button>
      <button (click)="selectAll()">Select All</button>
    </div>
    
    <div class="list">
      @for (item of items(); track item.id) {
        <app-list-item 
          [item]="item"
          (toggle)="toggleItem($event)">
        </app-list-item>
      }
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PerformanceDemoComponent {
  items = signal<any[]>([]);
  
  selectedCount = computed(() => 
    this.items().filter(i => i.selected).length
  );
  
  generateItems() {
    console.time('generate');
    const newItems = Array.from({ length: 10000 }, (_, i) => ({
      id: i,
      name: `Item ${i}`,
      selected: false
    }));
    this.items.set(newItems);
    console.timeEnd('generate');
  }
  
  selectAll() {
    console.time('select-all');
    this.items.update(items => 
      items.map(item => ({ ...item, selected: true }))
    );
    console.timeEnd('select-all');
    // With OnPush + signals: fast
    // With Default: slow
  }
  
  toggleItem(id: number) {
    this.items.update(items =>
      items.map(item =>
        item.id === id
          ? { ...item, selected: !item.selected }
          : item
      )
    );
  }
}

@Component({
  selector: 'app-list-item',
  template: `
    <div (click)="toggle.emit(item.id)">
      {{ item.name }} - {{ item.selected ? 'Selected' : 'Not Selected' }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListItemComponent {
  @Input() item: any;
  @Output() toggle = new EventEmitter<number>();
}
```

## Common Mistakes

### Mistake 1: Mutating Input Objects

```typescript
// BAD: Mutation doesn't trigger OnPush
@Component({
  selector: 'app-bad-mutation',
  template: `<div>{{ user.name }} - {{ user.age }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class BadMutationComponent {
  @Input() user!: User;
}

// Parent
@Component({
  template: `
    <app-bad-mutation [user]="user"></app-bad-mutation>
    <button (click)="updateUser()">Update</button>
  `
})
export class ParentComponent {
  user = { name: 'John', age: 30 };
  
  updateUser() {
    // Doesn't work with OnPush!
    this.user.age = 31;
  }
}

// GOOD: Create new reference
@Component({
  template: `
    <app-good-mutation [user]="user"></app-good-mutation>
    <button (click)="updateUser()">Update</button>
  `
})
export class ParentComponentGood {
  user = { name: 'John', age: 30 };
  
  updateUser() {
    // Works with OnPush
    this.user = { ...this.user, age: 31 };
  }
}
```

### Mistake 2: Forgetting markForCheck

```typescript
// BAD: Async updates without markForCheck
@Component({
  selector: 'app-forgot-markforcheck',
  template: `<div>Data: {{ data }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ForgotMarkForCheckComponent {
  data = '';
  
  ngOnInit() {
    // View won't update!
    setTimeout(() => {
      this.data = 'loaded';
    }, 1000);
  }
}

// GOOD: Use markForCheck
@Component({
  selector: 'app-with-markforcheck',
  template: `<div>Data: {{ data }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class WithMarkForCheckComponent {
  data = '';
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    setTimeout(() => {
      this.data = 'loaded';
      this.cdr.markForCheck(); // Required
    }, 1000);
  }
}

// BEST: Use signals
@Component({
  selector: 'app-with-signals',
  template: `<div>Data: {{ data() }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class WithSignalsComponent {
  data = signal('');
  
  ngOnInit() {
    setTimeout(() => {
      this.data.set('loaded'); // Automatic
    }, 1000);
  }
}
```

### Mistake 3: Using OnPush Without Immutability

```typescript
// BAD: OnPush with mutable operations
@Component({
  selector: 'app-mutable-list',
  template: `
    <div *ngFor="let item of items">{{ item }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class MutableListComponent {
  @Input() items: string[] = [];
}

// Parent
@Component({
  template: `
    <app-mutable-list [items]="items"></app-mutable-list>
    <button (click)="addItem()">Add</button>
  `
})
export class ParentComponent {
  items = ['a', 'b', 'c'];
  
  addItem() {
    // Doesn't trigger OnPush!
    this.items.push('d');
  }
}

// GOOD: Immutable operations
@Component({
  template: `
    <app-immutable-list [items]="items"></app-immutable-list>
    <button (click)="addItem()">Add</button>
  `
})
export class ParentComponentGood {
  items = ['a', 'b', 'c'];
  
  addItem() {
    // Creates new reference - triggers OnPush
    this.items = [...this.items, 'd'];
  }
}
```

## Best Practices

### 1. Always Use OnPush

```typescript
// Make OnPush the default for all components
@Component({
  selector: 'app-any-component',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AnyComponent {
  // OnPush should be your default strategy
}
```

### 2. Combine OnPush with Signals

```typescript
@Component({
  selector: 'app-modern-component',
  template: `
    <div>{{ state() | json }}</div>
    <button (click)="update()">Update</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ModernComponent {
  state = signal({ count: 0, name: 'Test' });
  
  update() {
    this.state.update(s => ({
      ...s,
      count: s.count + 1
    }));
  }
}
```

### 3. Use Async Pipe or toSignal

```typescript
@Component({
  selector: 'app-reactive-component',
  template: `
    <div>Observable: {{ data$ | async | json }}</div>
    <div>Signal: {{ dataSignal() | json }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ReactiveComponent {
  data$ = this.http.get('/api/data');
  dataSignal = toSignal(this.data$, { initialValue: null });
  
  constructor(private http: HttpClient) {}
}
```

### 4. Immutable Data Patterns

```typescript
@Component({
  selector: 'app-immutable-demo',
  template: `
    <app-child [user]="user()"></app-child>
    <button (click)="updateUser()">Update</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ImmutableDemoComponent {
  user = signal({ name: 'John', age: 30, address: { city: 'NYC' } });
  
  updateUser() {
    // Always create new references
    this.user.update(u => ({
      ...u,
      age: u.age + 1,
      address: {
        ...u.address,
        city: 'LA'
      }
    }));
  }
}
```

## Interview Questions

### Q1: What is OnPush change detection and how does it differ from Default?

**Answer:** OnPush is a change detection strategy where Angular only checks a component when: 1) an input property reference changes, 2) an event originates from the component, 3) the async pipe emits, or 4) change detection is manually triggered. Default strategy checks every component on every change detection cycle throughout the entire application, which is much more expensive.

### Q2: Why does OnPush require immutable data patterns?

**Answer:** OnPush uses reference equality to detect changes. If you mutate an object, its reference stays the same, so OnPush won't detect the change. You must create new object references (using spread operator, Object.assign, etc.) to trigger OnPush change detection.

### Q3: How do Signals work with OnPush?

**Answer:** Signals automatically trigger OnPush change detection when they update. Signal updates call markForCheck internally, so you don't need manual change detection. This makes Signals the perfect primitive for OnPush components.

### Q4: When should you manually call markForCheck()?

**Answer:** Call markForCheck when using OnPush with:

- Manual observable subscriptions (not async pipe)
- Async callbacks (setTimeout, setInterval)
- External library callbacks
- WebSocket messages
- Any async operation that updates component state outside Angular's event system

### Q5: Does the async pipe work with OnPush?

**Answer:** Yes, the async pipe is specifically designed to work with OnPush. It internally calls markForCheck() whenever the observable emits, ensuring the view updates correctly even with OnPush strategy.

## Key Takeaways

1. OnPush dramatically improves performance by reducing change detection cycles
2. OnPush requires immutable data patterns - create new references, don't mutate
3. Signals automatically work with OnPush, making them the ideal state primitive
4. The async pipe handles markForCheck automatically for observables
5. Always use ChangeDetectionStrategy.OnPush as your default strategy
6. OnPush checks when: input reference changes, component events, async pipe, or manual trigger
7. toSignal converts observables to signals for better OnPush integration
8. Manual subscriptions require manual markForCheck calls
9. OnPush is essential preparation for zoneless Angular
10. Combine OnPush + Signals for optimal performance

## Resources

### Official Documentation

- [Change Detection Guide](https://angular.dev/best-practices/runtime-performance)
- [OnPush API Reference](https://angular.dev/api/core/ChangeDetectionStrategy)
- [Signals Documentation](https://angular.dev/guide/signals)

### Articles

- [Understanding OnPush](https://blog.angular.io/understanding-angular-change-detection)
- [OnPush Best Practices](https://angular.dev/guide/change-detection)
- [Immutable Patterns in Angular](https://blog.angular.io/immutable-patterns)

### Tools

- Angular DevTools for change detection profiling
- Chrome Performance tab
- angular-profiler for detailed CD analysis

### Videos

- "OnPush Deep Dive" - Angular official
- "Performance Optimization" conference talks
