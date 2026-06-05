# Zone.js & Monkey Patching

## The Idea

**In plain English:** Zone.js is a tool that secretly wraps JavaScript's built-in timers and event listeners so it always knows when something async (like a timer or a network request) finishes. "Monkey patching" just means quietly swapping out the real function for a custom one that does extra work behind the scenes, without anyone asking for permission.

**Real-world analogy:** Imagine a hotel where every room phone is secretly replaced by the front desk with a special phone that looks identical but automatically notifies the front desk whenever a guest makes or receives a call — without the guest knowing the phone was swapped.

- The original room phone = the native browser API (e.g., `setTimeout`)
- The replacement special phone = Zone.js's patched version of that API
- The notification to the front desk = Zone.js triggering Angular's change detection after the async task finishes

---

## Overview

Zone.js is a library that provides execution contexts (zones) for asynchronous operations. Angular uses Zone.js to automatically trigger change detection after async operations complete. Understanding how Zone.js works through monkey patching, its architecture, task tracking, and performance implications reveals how Angular achieves automatic view updates.

This guide explores Zone.js internals, the monkey patching mechanism, zone forking, task interception, and running code outside Angular's zone for optimization.

## What is Zone.js

Zone.js creates execution contexts that persist across async operations:

```typescript
// Example 1: Basic zone concept
// Without Zone.js:
console.log('Before timeout');
setTimeout(() => {
  console.log('Inside timeout');
  // Lost context - no way to track this belongs to original operation
}, 1000);

// With Zone.js:
const myZone = Zone.current.fork({
  name: 'myZone',
  onInvokeTask: (delegate, current, target, task, applyThis, applyArgs) => {
    console.log('Task starting:', task.source);
    const result = delegate.invokeTask(target, task, applyThis, applyArgs);
    console.log('Task completed:', task.source);
    return result;
  }
});

myZone.run(() => {
  console.log('Before timeout');
  setTimeout(() => {
    console.log('Inside timeout');
    // Still in myZone context!
  }, 1000);
});

// Output:
// Before timeout
// Task starting: setTimeout
// Inside timeout
// Task completed: setTimeout
```

## Monkey Patching Mechanism

Zone.js patches global APIs to intercept async operations:

```typescript
// Example 2: How setTimeout is patched
// Original setTimeout
const nativeSetTimeout = window.setTimeout;

// Zone.js patches it
window.setTimeout = function(callback, delay, ...args) {
  // Get current zone
  const currentZone = Zone.current;
  
  // Wrap callback to run in zone
  const wrappedCallback = function() {
    return currentZone.runGuarded(callback, this, args);
  };
  
  // Schedule task
  const task = {
    source: 'setTimeout',
    callback: wrappedCallback,
    zone: currentZone,
    type: 'macroTask'
  };
  
  // Invoke lifecycle hooks
  currentZone.scheduleTask(task);
  
  // Call original setTimeout with wrapped callback
  const id = nativeSetTimeout.call(
    window,
    wrappedCallback,
    delay,
    ...args
  );
  
  task.data = { handleId: id };
  return id;
};

// Example 3: Patched APIs
// Zone.js patches these globals:
// - setTimeout, setInterval, setImmediate
// - requestAnimationFrame
// - addEventListener, removeEventListener
// - Promise
// - XMLHttpRequest, fetch
// - MutationObserver, IntersectionObserver
// - Node.js APIs (fs, crypto, etc. in server-side)

// Example 4: Event listener patching
const nativeAddEventListener = EventTarget.prototype.addEventListener;

EventTarget.prototype.addEventListener = function(
  type: string,
  listener: EventListener,
  options?: AddEventListenerOptions
) {
  const currentZone = Zone.current;
  
  // Wrap listener to run in zone
  const wrappedListener = function(event: Event) {
    return currentZone.runGuarded(listener, this, [event]);
  };
  
  // Store wrapped listener for removal
  (listener as any).__zone_symbol__wrappedListener = wrappedListener;
  
  // Call native addEventListener with wrapped listener
  return nativeAddEventListener.call(this, type, wrappedListener, options);
};
```

## Zone Architecture

```typescript
// Example 5: Zone structure
interface Zone {
  name: string;
  parent: Zone | null;
  
  // Properties (inherited from parent)
  get(key: string): any;
  
  // Fork creates child zone
  fork(zoneSpec: ZoneSpec): Zone;
  
  // Run callback in this zone
  run<T>(callback: () => T, applyThis?: any, applyArgs?: any[]): T;
  
  // Run with error handling
  runGuarded<T>(callback: () => T, applyThis?: any, applyArgs?: any[]): T;
  
  // Task scheduling
  scheduleTask<T extends Task>(task: T): T;
  invokeTask<T extends Task>(task: T, applyThis?: any, applyArgs?: any[]): any;
  cancelTask(task: Task): void;
}

// Example 6: Zone specification
interface ZoneSpec {
  name: string;
  properties?: { [key: string]: any };
  
  // Lifecycle hooks
  onFork?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, zoneSpec: ZoneSpec) => Zone;
  onIntercept?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, delegate: Function, source: string) => Function;
  onInvoke?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, delegate: Function, applyThis: any, applyArgs?: any[], source?: string) => any;
  onHandleError?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, error: Error) => boolean;
  onScheduleTask?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, task: Task) => Task;
  onInvokeTask?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, task: Task, applyThis: any, applyArgs?: any[]) => any;
  onCancelTask?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, task: Task) => any;
  onHasTask?: (parentZoneDelegate: ZoneDelegate, currentZone: Zone, targetZone: Zone, hasTaskState: HasTaskState) => void;
}

// Example 7: Creating custom zone
const customZone = Zone.current.fork({
  name: 'customZone',
  
  properties: {
    userId: 123,
    requestId: 'abc'
  },
  
  onInvoke(delegate, current, target, callback, applyThis, applyArgs) {
    console.log(`[${target.name}] Invoking:`, callback.name);
    return delegate.invoke(target, callback, applyThis, applyArgs);
  },
  
  onHandleError(delegate, current, target, error) {
    console.error(`[${target.name}] Error:`, error);
    // Return false to propagate, true to handle
    return false;
  }
});

customZone.run(() => {
  console.log('User ID:', Zone.current.get('userId'));
  // Access to properties
  
  setTimeout(() => {
    console.log('Still in custom zone');
    console.log('User ID:', Zone.current.get('userId'));
  }, 1000);
});
```

## Task Types

```typescript
// Example 8: Task classification
enum TaskType {
  microTask,    // Promise.then, queueMicrotask
  macroTask,    // setTimeout, setInterval, requestAnimationFrame
  eventTask     // addEventListener
}

// Example 9: MicroTask (Promise)
const nativePromiseThen = Promise.prototype.then;

Promise.prototype.then = function(onFulfilled, onRejected) {
  const currentZone = Zone.current;
  
  const wrappedOnFulfilled = onFulfilled && function(value: any) {
    return currentZone.runGuarded(onFulfilled, this, [value]);
  };
  
  const wrappedOnRejected = onRejected && function(error: any) {
    return currentZone.runGuarded(onRejected, this, [error]);
  };
  
  // Create micro task
  const task = {
    type: TaskType.microTask,
    source: 'Promise.then',
    zone: currentZone
  };
  
  currentZone.scheduleTask(task);
  
  return nativePromiseThen.call(
    this,
    wrappedOnFulfilled,
    wrappedOnRejected
  );
};

// Example 10: MacroTask (setTimeout)
// Already shown in Example 2

// Example 11: EventTask (addEventListener)
// Already shown in Example 4
```

## Angular's NgZone

Angular wraps Zone.js in NgZone service:

```typescript
// Example 12: NgZone implementation (simplified)
@Injectable({ providedIn: 'root' })
export class NgZone {
  private outer: Zone = Zone.current;
  private inner: Zone;
  
  // Emits when Angular zone runs
  readonly onMicrotaskEmpty = new EventEmitter<void>();
  readonly onStable = new EventEmitter<void>();
  readonly onUnstable = new EventEmitter<void>();
  readonly onError = new EventEmitter<Error>();
  
  constructor() {
    // Fork Angular zone
    this.inner = this.outer.fork({
      name: 'angular',
      
      properties: {
        isAngularZone: true
      },
      
      onInvokeTask: (delegate, current, target, task, applyThis, applyArgs) => {
        try {
          this.onEnter();
          return delegate.invokeTask(target, task, applyThis, applyArgs);
        } finally {
          this.onLeave();
        }
      },
      
      onHasTask: (delegate, current, target, hasTaskState) => {
        delegate.hasTask(target, hasTaskState);
        
        if (!hasTaskState.macroTask && !hasTaskState.microTask) {
          // No tasks left
          this.onMicrotaskEmpty.emit();
          this.checkStable();
        }
      },
      
      onHandleError: (delegate, current, target, error) => {
        this.onError.emit(error);
        return false;
      }
    });
  }
  
  // Run callback inside Angular zone
  run<T>(fn: (...args: any[]) => T, applyThis?: any, applyArgs?: any[]): T {
    return this.inner.run(fn, applyThis, applyArgs);
  }
  
  // Run callback outside Angular zone (no change detection)
  runOutsideAngular<T>(fn: (...args: any[]) => T): T {
    return this.outer.run(fn);
  }
  
  private onEnter() {
    this.onUnstable.emit();
  }
  
  private onLeave() {
    this.checkStable();
  }
  
  private checkStable() {
    if (!this.hasPendingTasks()) {
      this.onStable.emit();
      // Trigger change detection
    }
  }
}

// Example 13: Using NgZone
@Component({
  selector: 'app-zone-demo',
  template: `
    <div>{{ count }}</div>
    <button (click)="startTimer()">Start</button>
  `
})
export class ZoneDemoComponent {
  count = 0;
  
  constructor(private ngZone: NgZone) {
    // Listen to zone events
    this.ngZone.onStable.subscribe(() => {
      console.log('Zone stable - change detection triggered');
    });
  }
  
  startTimer() {
    this.ngZone.runOutsideAngular(() => {
      // This interval won't trigger change detection
      setInterval(() => {
        this.count++;
        console.log('Count:', this.count);
        // View doesn't update
        
        if (this.count % 10 === 0) {
          // Periodically run in Angular zone
          this.ngZone.run(() => {
            console.log('Updating view');
          });
        }
      }, 100);
    });
  }
}
```

## Running Outside Angular

```typescript
// Example 14: Performance optimization
@Component({
  selector: 'app-scroll-handler',
  template: `
    <div class="scrollable" (scroll)="onScroll($event)">
      <div class="content" [style.height.px]="contentHeight"></div>
    </div>
    <div>Scroll position: {{ scrollPosition }}</div>
  `
})
export class ScrollHandlerComponent {
  scrollPosition = 0;
  contentHeight = 10000;
  
  constructor(
    private ngZone: NgZone,
    private cdr: ChangeDetectorRef
  ) {}
  
  onScroll(event: Event) {
    // BAD: Triggers change detection on every scroll event (60fps)
    // this.scrollPosition = (event.target as Element).scrollTop;
    
    // GOOD: Run outside Angular
    this.ngZone.runOutsideAngular(() => {
      const scrollTop = (event.target as Element).scrollTop;
      this.scrollPosition = scrollTop;
      
      // Manually update view when needed
      if (scrollTop % 100 === 0) {
        this.ngZone.run(() => {
          this.cdr.detectChanges();
        });
      }
    });
  }
}

// Example 15: Animation optimization
@Component({
  selector: 'app-animation',
  template: `<canvas #canvas></canvas>`
})
export class AnimationComponent implements OnInit {
  @ViewChild('canvas') canvas: ElementRef<HTMLCanvasElement>;
  
  constructor(private ngZone: NgZone) {}
  
  ngOnInit() {
    this.ngZone.runOutsideAngular(() => {
      this.animate();
    });
  }
  
  private animate() {
    const ctx = this.canvas.nativeElement.getContext('2d');
    let x = 0;
    
    const render = () => {
      ctx.clearRect(0, 0, 300, 300);
      ctx.fillRect(x, 100, 50, 50);
      x = (x + 1) % 300;
      
      // 60fps, no change detection
      requestAnimationFrame(render);
    };
    
    requestAnimationFrame(render);
  }
}

// Example 16: Third-party library integration
@Component({
  selector: 'app-chart',
  template: `<div #chartContainer></div>`
})
export class ChartComponent implements OnInit, OnDestroy {
  @ViewChild('chartContainer') container: ElementRef;
  private chart: any;
  
  constructor(private ngZone: NgZone) {}
  
  ngOnInit() {
    // Initialize chart outside Angular
    this.ngZone.runOutsideAngular(() => {
      this.chart = new ThirdPartyChart(this.container.nativeElement, {
        data: this.data,
        onUpdate: (newData) => {
          // Library callback runs outside Angular
          // Manually trigger change detection if needed
          this.ngZone.run(() => {
            this.handleUpdate(newData);
          });
        }
      });
    });
  }
  
  ngOnDestroy() {
    this.ngZone.runOutsideAngular(() => {
      this.chart.destroy();
    });
  }
}
```

## Zone.js Configuration

```typescript
// Example 17: Disabling specific patches
// In polyfills.ts before zone.js import:

// Disable requestAnimationFrame patch
(window as any).__Zone_disable_requestAnimationFrame = true;

// Disable all browser patches
(window as any).__Zone_disable_IE_check = true;
(window as any).__Zone_disable_on_property = true;

// Then import zone.js
import 'zone.js';

// Example 18: Custom zone configuration
// zone-flags.ts
(window as any).__zone_symbol__UNPATCHED_EVENTS = ['scroll', 'mousemove'];

// These events won't trigger change detection

// Example 19: Checking if in Angular zone
function isInAngularZone(): boolean {
  return Zone.current.get('isAngularZone') === true;
}

// Use in services:
@Injectable()
export class MyService {
  doWork() {
    if (!isInAngularZone()) {
      console.warn('Not in Angular zone, change detection won\'t run');
    }
  }
}
```

## Performance Implications

```typescript
// Example 20: Measuring zone overhead
@Component({
  selector: 'app-benchmark',
  template: `<div>{{ result }}</div>`
})
export class BenchmarkComponent {
  result = '';
  
  constructor(private ngZone: NgZone) {}
  
  benchmark() {
    // With zone
    const withZoneStart = performance.now();
    for (let i = 0; i < 1000; i++) {
      setTimeout(() => {}, 0);
    }
    const withZoneEnd = performance.now();
    
    // Without zone
    this.ngZone.runOutsideAngular(() => {
      const withoutZoneStart = performance.now();
      for (let i = 0; i < 1000; i++) {
        setTimeout(() => {}, 0);
      }
      const withoutZoneEnd = performance.now();
      
      this.result = `
        With zone: ${withZoneEnd - withZoneStart}ms
        Without zone: ${withoutZoneEnd - withoutZoneStart}ms
      `;
    });
  }
}
```

## Common Misconceptions

1. **"Zone.js is required for Angular"** - False. Angular can run zoneless with manual change detection triggers.

2. **"runOutsideAngular prevents all change detection"** - False. It only prevents automatic triggering. Manual detectChanges() still works.

3. **"Zone.js makes Angular slow"** - Mostly false. Zone.js overhead is minimal. Most performance issues are from excessive change detection, not Zone.js itself.

4. **"All async operations trigger change detection"** - True by default, but you can disable patches or run outside Angular.

5. **"Zone.js only works in browsers"** - False. It works in Node.js too, patching relevant APIs.

## Performance Implications

1. **Patching Overhead** - Wrapping functions adds slight overhead to async operations, usually negligible.

2. **Change Detection Triggering** - Zone.js triggers change detection after every async operation, which can be excessive for high-frequency events.

3. **Memory** - Zone.js maintains task queues and context, adding memory overhead.

4. **Optimization** - Use runOutsideAngular for high-frequency operations that don't need change detection.

## Interview Questions

1. **Q: What is Zone.js and why does Angular use it?**
   A: Zone.js provides execution contexts that persist across async operations. Angular uses it to automatically trigger change detection after async events complete, enabling automatic view updates without manual intervention.

2. **Q: How does Zone.js monkey patch async APIs?**
   A: Zone.js replaces global async APIs (setTimeout, Promise, addEventListener, etc.) with wrapped versions. These wrappers capture the current zone, execute callbacks within that zone, and invoke lifecycle hooks (like onInvokeTask).

3. **Q: What is the difference between run() and runOutsideAngular()?**
   A: run() executes code inside Angular's zone, triggering change detection after completion. runOutsideAngular() executes code outside Angular's zone, preventing automatic change detection, useful for performance optimization.

4. **Q: What are the three types of tasks in Zone.js?**
   A: MicroTasks (Promise.then, queueMicrotask), MacroTasks (setTimeout, setInterval, requestAnimationFrame), and EventTasks (addEventListener). Each has different scheduling and execution characteristics.

5. **Q: How can you disable Zone.js for specific events?**
   A: Set global flags before importing Zone.js, like __zone_symbol__UNPATCHED_EVENTS = ['scroll', 'mousemove']. These events won't trigger change detection automatically.

6. **Q: What is NgZone and how does it differ from Zone.js?**
   A: NgZone is Angular's service wrapping Zone.js functionality. It creates the 'angular' zone, provides convenient methods (run, runOutsideAngular), and emits events (onStable, onUnstable) that Angular uses to coordinate change detection.

7. **Q: When would you use runOutsideAngular()?**
   A: For high-frequency operations that don't need change detection: animations, scroll handlers, mouse move tracking, or third-party library integrations. This prevents excessive change detection cycles.

8. **Q: How does Zone.js handle error propagation?**
   A: Zone.js catches errors in async operations and invokes the onHandleError hook. If the hook returns false, the error propagates. Angular's zone catches these errors and logs them to the console.

## Key Takeaways

1. Zone.js provides execution contexts that persist across async operations
2. It monkey patches global APIs to intercept and wrap callbacks
3. Angular uses Zone.js to automatically trigger change detection
4. Three task types: micro, macro, and event tasks
5. NgZone wraps Zone.js with Angular-specific functionality
6. runOutsideAngular prevents automatic change detection for performance
7. Zone.js can be configured to disable specific patches
8. Understanding zones is crucial for optimizing Angular performance

## Resources

- [Zone.js GitHub](https://github.com/angular/angular/tree/main/packages/zone.js)
- [Understanding Zone.js](https://blog.thoughtram.io/angular/2016/02/01/zones-in-angular-2.html)
- [NgZone API Documentation](https://angular.io/api/core/NgZone)
- [Zone.js Best Practices](https://blog.angular-university.io/zone-js/)
- [Angular Without Zone.js (Zoneless)](https://angular.io/guide/change-detection-zone-pollution)
