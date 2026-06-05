# Change Detection Algorithm

## The Idea

**In plain English:** Change detection is how a web framework (like Angular) automatically notices when your data changes and updates what the user sees on screen. Think of a "framework" as a helper that builds and manages your webpage, and "data" as the information your app stores, like a username or a list of items.

**Real-world analogy:** Imagine a school office assistant whose job is to walk through every classroom at the end of each period and check whether the whiteboard still matches the lesson plan on file. If something is different, they update the record.

- The assistant doing the walkthrough = Angular's change detection running through the component tree
- Each classroom = a component (a self-contained piece of the webpage, like a header or a button)
- The lesson plan on file = the previous stored value of a piece of data
- The current whiteboard content = the new current value of that data
- Noticing a mismatch and updating the record = Angular detecting a change and re-rendering the screen

---

## Overview

Angular's change detection mechanism is responsible for keeping the view synchronized with the component model. Understanding how Angular traverses the component tree, when change detection runs, what triggers it, and how to optimize it is crucial for building performant Angular applications.

This guide explores the change detection tree, traversal algorithm, zones integration, and optimization strategies like OnPush and manual control.

## Change Detection Tree

Angular applications form a tree of components, each with its own change detector:

```typescript
// Example 1: Component tree structure
@Component({
  selector: 'app-root',
  template: `
    <app-header></app-header>
    <app-main>
      <app-sidebar></app-sidebar>
      <app-content></app-content>
    </app-main>
  `
})
export class AppComponent {}

// Internal structure (simplified):
class ChangeDetectorRef {
  context: Component;
  parent: ChangeDetectorRef | null;
  children: ChangeDetectorRef[];
  
  detectChanges() {
    // Check bindings for this component
    this.checkBindings();
    
    // Recurse to children
    this.children.forEach(child => child.detectChanges());
  }
}

// Tree:
// AppComponent (CD)
//   ├─ HeaderComponent (CD)
//   └─ MainComponent (CD)
//       ├─ SidebarComponent (CD)
//       └─ ContentComponent (CD)
```

### Change Detection States

```typescript
// Example 2: Change detector states
enum ChangeDetectorStatus {
  CheckOnce,      // Check once, then detach
  Checked,        // Already checked this cycle
  CheckAlways,    // Always check (default)
  Detached,       // Detached from tree
  Errored,        // Error occurred
  Destroyed       // Component destroyed
}

// Example 3: Component with state
@Component({
  selector: 'app-user',
  template: `
    <div>{{ user.name }}</div>
    <div>{{ user.email }}</div>
  `
})
export class UserComponent {
  @Input() user: User;
  
  // Change detector tracks:
  // - Previous values of bindings
  // - Current values of bindings
  // - State (CheckAlways, Detached, etc.)
}
```

## The Algorithm

The core algorithm compares previous and current values:

```typescript
// Example 4: Simplified change detection
function detectChanges(component: Component, view: View) {
  // 1. Update input bindings
  updateInputProperties(component, view);
  
  // 2. Execute lifecycle hooks
  if (component.ngDoCheck) {
    component.ngDoCheck();
  }
  
  // 3. Check child component inputs
  checkChildComponentInputs(view);
  
  // 4. Check content children
  checkContentChildren(view);
  
  // 5. Update view bindings (template expressions)
  updateViewBindings(view);
  
  // 6. Check view children
  checkViewChildren(view);
  
  // 7. Execute lifecycle hooks
  if (component.ngAfterViewChecked) {
    component.ngAfterViewChecked();
  }
}

// Example 5: Binding check
function checkBinding(
  view: View,
  bindingIndex: number,
  value: any
): boolean {
  const oldValue = view.oldValues[bindingIndex];
  
  if (oldValue !== value) {
    view.oldValues[bindingIndex] = value;
    return true; // Changed
  }
  
  return false; // Unchanged
}

// Example 6: View update
@Component({
  selector: 'app-counter',
  template: `
    <div>{{ count }}</div>
    <div>{{ doubled }}</div>
    <button (click)="increment()">Increment</button>
  `
})
export class CounterComponent {
  count = 0;
  
  get doubled() {
    console.log('Getter called');
    return this.count * 2;
  }
  
  increment() {
    this.count++;
    // Triggers change detection
  }
}

// During change detection:
// 1. Check binding 0: count (0 → 1, changed)
// 2. Update DOM for count
// 3. Check binding 1: doubled (calls getter, 0 → 2, changed)
// 4. Update DOM for doubled
```

## Tree Traversal

Angular traverses the component tree depth-first, top-down:

```typescript
// Example 7: Traversal order
@Component({
  selector: 'app-root',
  template: `
    <app-a></app-a>
    <app-b>
      <app-c></app-c>
      <app-d></app-d>
    </app-b>
    <app-e></app-e>
  `
})
export class AppComponent {}

// Traversal order:
// 1. AppComponent
// 2. AppA
// 3. AppB
// 4. AppC
// 5. AppD
// 6. AppE

// Example 8: Simplified traversal implementation
function checkAndUpdateView(view: View) {
  // Check current component
  if (view.state !== ChangeDetectorStatus.Detached) {
    checkAndUpdateComponent(view);
  }
  
  // Check children
  for (const childView of view.children) {
    checkAndUpdateView(childView);
  }
  
  // Check projected content
  for (const contentView of view.contentChildren) {
    checkAndUpdateView(contentView);
  }
}
```

### Skipping Subtrees

```typescript
// Example 9: OnPush strategy skips subtrees
@Component({
  selector: 'app-expensive',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ data.value }}</div>
    <app-child [data]="childData"></app-child>
  `
})
export class ExpensiveComponent {
  @Input() data: Data;
  childData = { value: 42 };
}

// With OnPush:
// - Only checks when @Input() reference changes
// - Or when event fires inside component
// - Or when async pipe receives new value
// - Entire subtree skipped if no changes

// Example 10: Marking for check
@Component({
  selector: 'app-manual',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div>{{ value }}</div>`
})
export class ManualComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateValue() {
    this.value++;
    // OnPush won't detect this mutation
    
    this.cdr.markForCheck();
    // Marks this component and ancestors for checking
    // Next change detection cycle will check this subtree
  }
}
```

## What Triggers Change Detection

```typescript
// Example 11: Event triggers
@Component({
  selector: 'app-events',
  template: `
    <button (click)="handleClick()">Click</button>
    <input (input)="handleInput($event)">
  `
})
export class EventsComponent {
  handleClick() {
    // Triggers change detection for entire tree
    console.log('Clicked');
  }
  
  handleInput(event: Event) {
    // Triggers change detection for entire tree
    console.log(event);
  }
}

// Example 12: Timer triggers
export class TimerComponent {
  count = 0;
  
  ngOnInit() {
    setInterval(() => {
      this.count++;
      // Triggers change detection every second
    }, 1000);
  }
}

// Example 13: HTTP triggers
export class DataComponent {
  users: User[] = [];
  
  constructor(private http: HttpClient) {}
  
  ngOnInit() {
    this.http.get<User[]>('/api/users').subscribe(users => {
      this.users = users;
      // Triggers change detection when response arrives
    });
  }
}

// Example 14: Manual triggers
export class ManualTriggerComponent {
  constructor(
    private cdr: ChangeDetectorRef,
    private appRef: ApplicationRef
  ) {}
  
  // Trigger for this component
  localUpdate() {
    this.cdr.detectChanges();
  }
  
  // Trigger for entire application
  globalUpdate() {
    this.appRef.tick();
  }
}
```

## Zone.js Integration

Zone.js patches async APIs to trigger change detection:

```typescript
// Example 15: How Zone.js works
// Zone.js patches setTimeout:
const originalSetTimeout = window.setTimeout;

window.setTimeout = function(callback: Function, delay: number) {
  return originalSetTimeout(() => {
    callback();
    // After callback, trigger change detection
    Zone.current.get('ng').run(() => {
      applicationRef.tick();
    });
  }, delay);
};

// Example 16: Running outside Angular
@Component({
  selector: 'app-outside',
  template: `<div>{{ count }}</div>`
})
export class OutsideComponent {
  count = 0;
  
  constructor(private ngZone: NgZone) {}
  
  startCounter() {
    // Run outside Angular's zone
    this.ngZone.runOutsideAngular(() => {
      setInterval(() => {
        this.count++;
        // Change detection NOT triggered
        // View won't update
      }, 1000);
    });
  }
  
  startCounterInZone() {
    this.ngZone.runOutsideAngular(() => {
      setInterval(() => {
        this.count++;
        
        // Manually trigger change detection
        this.ngZone.run(() => {
          // Now change detection runs
        });
      }, 1000);
    });
  }
}
```

## OnPush Change Detection

OnPush strategy only checks when inputs change or events fire:

```typescript
// Example 17: OnPush component
@Component({
  selector: 'app-onpush',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ user.name }}</div>
    <button (click)="changeName()">Change</button>
  `
})
export class OnPushComponent {
  @Input() user: User;
  
  changeName() {
    // Won't trigger update (mutation)
    this.user.name = 'New Name';
  }
  
  changeNameCorrect() {
    // Will trigger update (new reference)
    this.user = { ...this.user, name: 'New Name' };
  }
}

// Example 18: OnPush with async pipe
@Component({
  selector: 'app-async',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ user$ | async }}</div>
  `
})
export class AsyncComponent {
  user$ = this.http.get<User>('/api/user');
  
  constructor(private http: HttpClient) {}
  
  // Async pipe automatically:
  // 1. Subscribes to observable
  // 2. Calls markForCheck() when value emits
  // 3. Unsubscribes on destroy
}

// Example 19: OnPush gotchas
@Component({
  selector: 'app-onpush-gotcha',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ items.length }} items</div>
    <div *ngFor="let item of items">{{ item }}</div>
  `
})
export class OnPushGotchaComponent {
  @Input() items: string[];
  
  addItem(item: string) {
    // Won't update view (array mutated, reference unchanged)
    this.items.push(item);
  }
  
  addItemCorrect(item: string) {
    // Will update view (new array reference)
    this.items = [...this.items, item];
  }
}
```

## Detaching and Reattaching

```typescript
// Example 20: Manual control
@Component({
  selector: 'app-manual-control',
  template: `
    <div>{{ value }}</div>
    <button (click)="update()">Update</button>
  `
})
export class ManualControlComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {
    // Detach from change detection tree
    this.cdr.detach();
  }
  
  update() {
    this.value++;
    
    // Manually trigger check
    this.cdr.detectChanges();
    // Or reattach to tree
    // this.cdr.reattach();
  }
}

// Example 21: Optimizing high-frequency updates
@Component({
  selector: 'app-chart',
  template: `<canvas #canvas></canvas>`
})
export class ChartComponent implements OnInit, OnDestroy {
  @ViewChild('canvas') canvas: ElementRef<HTMLCanvasElement>;
  
  constructor(
    private cdr: ChangeDetectorRef,
    private ngZone: NgZone
  ) {
    // Detach - we'll manage updates manually
    this.cdr.detach();
  }
  
  ngOnInit() {
    this.ngZone.runOutsideAngular(() => {
      // High-frequency updates outside Angular
      this.startAnimation();
    });
  }
  
  startAnimation() {
    const ctx = this.canvas.nativeElement.getContext('2d');
    
    const render = () => {
      // Draw directly to canvas
      // No change detection needed
      ctx.clearRect(0, 0, 300, 300);
      ctx.fillRect(Math.random() * 300, Math.random() * 300, 50, 50);
      
      requestAnimationFrame(render);
    };
    
    requestAnimationFrame(render);
  }
}
```

## Common Pitfalls

```typescript
// Example 22: Expression changed after check
@Component({
  selector: 'app-expr-changed',
  template: `<div>{{ value }}</div>`
})
export class ExprChangedComponent {
  _value = 0;
  
  get value() {
    // DON'T: Mutate state in getter
    return ++this._value;
    // Causes ExpressionChangedAfterItHasBeenCheckedError
  }
}

// Example 23: Correct approach
@Component({
  selector: 'app-expr-correct',
  template: `<div>{{ value }}</div>`
})
export class ExprCorrectComponent {
  value = 0;
  
  constructor() {
    // Mutate in constructor or lifecycle hook
    setTimeout(() => {
      this.value++;
    });
  }
}
```

## Performance Optimization

```typescript
// Example 24: trackBy for lists
@Component({
  selector: 'app-list',
  template: `
    <div *ngFor="let item of items; trackBy: trackById">
      {{ item.name }}
    </div>
  `
})
export class ListComponent {
  items: Item[] = [];
  
  trackById(index: number, item: Item) {
    return item.id;
    // Angular reuses DOM nodes for items with same ID
    // Without trackBy, recreates all nodes on array change
  }
}

// Example 25: Pure pipes
@Pipe({
  name: 'expensive',
  pure: true // Default, only called when input changes
})
export class ExpensivePipe implements PipeTransform {
  transform(value: string): string {
    console.log('Pipe called');
    return value.toUpperCase();
  }
}

// With pure: true
// Pipe only called when 'value' reference changes
// Cached result reused on subsequent checks

// Example 26: Immutable data patterns
@Component({
  selector: 'app-immutable',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let item of items">{{ item.name }}</div>
  `
})
export class ImmutableComponent {
  items: Item[] = [];
  
  addItem(item: Item) {
    // Create new array (OnPush detects change)
    this.items = [...this.items, item];
  }
  
  updateItem(id: number, updates: Partial<Item>) {
    // Create new array with updated object
    this.items = this.items.map(item =>
      item.id === id ? { ...item, ...updates } : item
    );
  }
  
  removeItem(id: number) {
    // Create new array without item
    this.items = this.items.filter(item => item.id !== id);
  }
}
```

## Common Misconceptions

1. **"Change detection runs on every tick"** - False. It runs when async events occur (events, timers, HTTP) or when manually triggered.

2. **"OnPush makes components faster"** - True, but only if used correctly with immutable data patterns. Improper use can cause bugs.

3. **"Zone.js is required for Angular"** - False. You can run Angular zoneless, but must manually trigger change detection.

4. **"Change detection is slow"** - Usually false. Angular's change detection is highly optimized. Performance issues often stem from component design, not the algorithm.

5. **"Detaching disables change detection forever"** - False. You can reattach or manually call detectChanges().

## Performance Implications

1. **Tree Size** - Change detection time is O(n) where n is number of components. Larger trees take longer.

2. **Binding Count** - Each template binding is checked. Components with many bindings are slower.

3. **OnPush Strategy** - Can dramatically reduce checks by skipping entire subtrees when inputs unchanged.

4. **Detaching** - Completely removes component from change detection, useful for high-frequency updates.

## Interview Questions

1. **Q: Explain how Angular's change detection works.**
   A: Angular traverses the component tree depth-first, checking each binding by comparing previous and current values. For each component, it updates inputs, runs lifecycle hooks, checks template bindings, and recurses to children. Zone.js triggers change detection on async events.

2. **Q: What is the difference between Default and OnPush change detection?**
   A: Default checks every component on every change detection cycle. OnPush only checks when @Input references change, events fire inside the component, async pipe emits, or markForCheck() is called. OnPush enables skipping entire subtrees.

3. **Q: When does Angular trigger change detection?**
   A: When async events occur: user events (clicks, inputs), timers (setTimeout, setInterval), HTTP responses, or manual triggers (detectChanges(), markForCheck(), ApplicationRef.tick()). Zone.js patches async APIs to trigger automatically.

4. **Q: What is markForCheck() and when would you use it?**
   A: markForCheck() marks a component and its ancestors for checking in the next change detection cycle. Use it with OnPush when data changes but the reference doesn't, or when updating from outside Angular's zone.

5. **Q: What causes ExpressionChangedAfterItHasBeenCheckedError?**
   A: This error occurs when a value checked during change detection changes during the same cycle, typically from mutating state in getters or child-to-parent communication during initialization. Fix by moving mutations to appropriate lifecycle hooks.

6. **Q: How does detaching a component affect change detection?**
   A: Detaching removes the component from the change detection tree. Angular won't check it automatically. You must manually call detectChanges() or reattach() to update the view. Useful for high-frequency updates or manual optimization.

7. **Q: What is the role of Zone.js in change detection?**
   A: Zone.js patches async APIs (setTimeout, addEventListener, etc.) to intercept callbacks and trigger change detection after they execute. This enables automatic view updates without manual intervention.

8. **Q: How does trackBy improve ngFor performance?**
   A: trackBy provides a unique identifier for list items. Without it, Angular destroys and recreates all DOM nodes when the array changes. With trackBy, Angular reuses nodes for items with the same ID, only adding, removing, or moving nodes as needed.

## Key Takeaways

1. Change detection traverses the component tree depth-first, top-down
2. Each binding is checked by comparing previous and current values
3. Zone.js patches async APIs to trigger change detection automatically
4. OnPush strategy skips checking when inputs unchanged
5. markForCheck() marks component and ancestors for checking
6. Detaching removes component from change detection tree
7. Immutable data patterns work best with OnPush
8. trackBy optimizes list rendering by reusing DOM nodes

## Resources

- [Angular Change Detection](https://angular.io/guide/change-detection)
- [Change Detection Strategy](https://angular.io/api/core/ChangeDetectionStrategy)
- [Everything You Need to Know About Change Detection](https://blog.angular-university.io/how-does-angular-2-change-detection-really-work/)
- [Angular In Depth: Change Detection](https://indepth.dev/posts/1053/everything-you-need-to-know-about-change-detection-in-angular)
- [Zone.js Documentation](https://github.com/angular/angular/tree/main/packages/zone.js)
