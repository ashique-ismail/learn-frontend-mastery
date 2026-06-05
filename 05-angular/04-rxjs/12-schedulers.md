# RxJS Schedulers

## The Idea

**In plain English:** A scheduler is a way to tell your code exactly when to run a task — right now, a tiny moment later, or synced with the screen refreshing. Think of it as a traffic controller that decides the order and timing of jobs waiting to be done.

**Real-world analogy:** Imagine a busy restaurant kitchen where the head chef decides when each dish gets cooked. Some orders go straight to the grill right now, some are queued up in a specific order, some wait until the current rush is over, and some are timed to come out exactly when a table is ready.

- The head chef = the scheduler
- Each dish order = a task (a piece of code to run)
- The timing rules (right now vs. after the rush) = the scheduler type (queueScheduler, asyncScheduler, etc.)

---

## Table of Contents

- [Introduction](#introduction)
- [What are Schedulers](#what-are-schedulers)
- [Scheduler Types](#scheduler-types)
- [asyncScheduler](#asyncscheduler)
- [queueScheduler](#queuescheduler)
- [asapScheduler](#asapscheduler)
- [animationFrameScheduler](#animationframescheduler)
- [Parameterizing Schedulers](#parameterizing-schedulers)
- [Testing with Schedulers](#testing-with-schedulers)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Schedulers in RxJS control **when** and **how** subscriptions are executed and when notifications are delivered. They provide fine-grained control over the timing and execution context of observables, making them essential for performance optimization, testing, and coordinating asynchronous operations.

Most developers can use RxJS effectively without ever explicitly using schedulers, but understanding them unlocks advanced patterns and optimizations, especially for animation, testing, and performance-critical applications.

## What are Schedulers

A scheduler is a data structure that decides when a task should execute and provides an execution context. Think of schedulers as task queues with different priority and timing strategies.

### Core Concepts

```typescript
// Without scheduler - synchronous
const source$ = of(1, 2, 3);
console.log('Before subscribe');
source$.subscribe(val => console.log('Value:', val));
console.log('After subscribe');

// Output:
// "Before subscribe"
// "Value: 1"
// "Value: 2"
// "Value: 3"
// "After subscribe"

// With asyncScheduler - asynchronous
const asyncSource$ = of(1, 2, 3, asyncScheduler);
console.log('Before subscribe');
asyncSource$.subscribe(val => console.log('Value:', val));
console.log('After subscribe');

// Output:
// "Before subscribe"
// "After subscribe"
// "Value: 1"
// "Value: 2"
// "Value: 3"
```

### Virtual Time

```typescript
import { TestScheduler } from 'rxjs/testing';

// Schedulers enable virtual time for testing
const testScheduler = new TestScheduler((actual, expected) => {
  expect(actual).toEqual(expected);
});

testScheduler.run(({ cold, expectObservable }) => {
  const source$ = cold('a-b-c|');
  expectObservable(source$).toBe('a-b-c|');
});
// Tests complete instantly, no actual waiting!
```

## Scheduler Types

### Overview

```typescript
import {
  asyncScheduler,
  queueScheduler,
  asapScheduler,
  animationFrameScheduler
} from 'rxjs';

// asyncScheduler - Uses setTimeout/setInterval (macrotask)
// queueScheduler - Synchronous iteration queue
// asapScheduler - Uses Promise.then (microtask)
// animationFrameScheduler - Uses requestAnimationFrame
```

### Comparison Table

| Scheduler | Timing | Use Case | Example |
| --------- | ------ | -------- | ------- |
| **queueScheduler** | Synchronous | Immediate iteration | Recursion control |
| **asapScheduler** | Microtask | Before next macrotask | High-priority async |
| **asyncScheduler** | Macrotask | After current execution | Timers, delays |
| **animationFrameScheduler** | Next frame | Smooth animations | DOM animations |

## asyncScheduler

Uses `setTimeout` and `setInterval` for scheduling. This is the default scheduler for time-based operators.

### Basic Usage

```typescript
import { asyncScheduler } from 'rxjs';

// Schedule a task
asyncScheduler.schedule(() => {
  console.log('Executed asynchronously');
});

console.log('Synchronous code');

// Output:
// "Synchronous code"
// "Executed asynchronously"
```

### With Delay

```typescript
import { asyncScheduler } from 'rxjs';

console.log('Start');

// Schedule with 1 second delay
asyncScheduler.schedule(() => {
  console.log('After 1 second');
}, 1000);

console.log('End');

// Output:
// "Start"
// "End"
// (1 second pause)
// "After 1 second"
```

### Recurring Tasks

```typescript
import { asyncScheduler } from 'rxjs';

let count = 0;

asyncScheduler.schedule(function(this: any, state?: number) {
  console.log('Count:', state);
  
  if (state < 5) {
    // Reschedule with incremented state
    this.schedule(state + 1, 1000);
  }
}, 0, count);

// Output (1 second intervals):
// "Count: 0"
// "Count: 1"
// "Count: 2"
// "Count: 3"
// "Count: 4"
// "Count: 5"
```

### Real-World Example: Polling with Scheduler

```typescript
import { asyncScheduler, Observable } from 'rxjs';

@Injectable()
export class PollingService {
  constructor(private http: HttpClient) {}

  poll(url: string, intervalMs: number): Observable<any> {
    return new Observable(observer => {
      const pollFn = () => {
        this.http.get(url).subscribe({
          next: data => {
            observer.next(data);
            // Schedule next poll
            subscription = asyncScheduler.schedule(pollFn, intervalMs);
          },
          error: err => observer.error(err)
        });
      };

      let subscription = asyncScheduler.schedule(pollFn, 0);

      // Cleanup
      return () => subscription?.unsubscribe();
    });
  }
}
```

### interval and timer Use asyncScheduler

```typescript
import { interval, timer } from 'rxjs';

// interval uses asyncScheduler by default
const interval$ = interval(1000);
// Equivalent to:
const manualInterval$ = interval(1000, asyncScheduler);

// timer also uses asyncScheduler
const timer$ = timer(2000);
// Equivalent to:
const manualTimer$ = timer(2000, asyncScheduler);
```

## queueScheduler

Executes tasks synchronously in a queue, one after another. Useful for preventing stack overflow in recursive operations.

### Basic Usage

```typescript
import { queueScheduler } from 'rxjs';

console.log('Before');

queueScheduler.schedule(() => console.log('Task 1'));
queueScheduler.schedule(() => console.log('Task 2'));
queueScheduler.schedule(() => console.log('Task 3'));

console.log('After');

// Output (all synchronous):
// "Before"
// "Task 1"
// "Task 2"
// "Task 3"
// "After"
```

### Preventing Stack Overflow

```typescript
import { queueScheduler } from 'rxjs';

// WITHOUT scheduler - causes stack overflow
function recursiveSynchronous(count: number) {
  if (count > 0) {
    console.log(count);
    recursiveSynchronous(count - 1); // Stack grows!
  }
}

// recursiveSynchronous(100000); // Stack overflow!

// WITH queueScheduler - no stack overflow
function recursiveWithScheduler(count: number) {
  if (count > 0) {
    console.log(count);
    queueScheduler.schedule(() => recursiveWithScheduler(count - 1));
  }
}

recursiveWithScheduler(100000); // Works!
```

### Real-World Example: Safe Recursive Observable

```typescript
import { queueScheduler, Observable } from 'rxjs';

@Injectable()
export class TreeTraversalService {
  traverseTree(node: TreeNode): Observable<TreeNode> {
    return new Observable(observer => {
      const traverse = (current: TreeNode) => {
        observer.next(current);

        // Schedule children traversal to prevent stack overflow
        current.children.forEach(child => {
          queueScheduler.schedule(() => traverse(child));
        });
      };

      traverse(node);
      observer.complete();
    });
  }
}
```

## asapScheduler

Uses `Promise.then()` to schedule tasks in the microtask queue. Executes before the next macrotask but after current synchronous code.

### Basic Usage

```typescript
import { asapScheduler } from 'rxjs';

console.log('Start');

asapScheduler.schedule(() => console.log('asap task'));

Promise.resolve().then(() => console.log('Promise'));

setTimeout(() => console.log('setTimeout'), 0);

console.log('End');

// Output:
// "Start"
// "End"
// "asap task"
// "Promise"
// "setTimeout"
```

### Priority Over setTimeout

```typescript
import { asapScheduler, asyncScheduler } from 'rxjs';

console.log('1. Synchronous');

setTimeout(() => console.log('4. setTimeout'), 0);

asapScheduler.schedule(() => console.log('2. asap'));

Promise.resolve().then(() => console.log('3. Promise'));

// Output:
// "1. Synchronous"
// "2. asap"
// "3. Promise"
// "4. setTimeout"
```

### Real-World Example: High-Priority Updates

```typescript
import { asapScheduler } from 'rxjs';

@Injectable()
export class NotificationService {
  private notifications: Notification[] = [];

  addNotification(notification: Notification): void {
    this.notifications.push(notification);

    // Process notification ASAP (before any setTimeout tasks)
    asapScheduler.schedule(() => {
      this.processNotification(notification);
    });
  }

  private processNotification(notification: Notification): void {
    console.log('Processing notification:', notification);
    // Update UI, send analytics, etc.
  }
}
```

## animationFrameScheduler

Uses `requestAnimationFrame()` to schedule tasks synchronized with browser repaints. Perfect for smooth animations and DOM updates.

### Basic Usage

```typescript
import { animationFrameScheduler } from 'rxjs';

let frame = 0;

const animate = () => {
  console.log('Frame:', frame++);
  
  if (frame < 60) {
    animationFrameScheduler.schedule(animate);
  }
};

animationFrameScheduler.schedule(animate);
// Runs at 60fps (or whatever the display refresh rate is)
```

### Smooth Animation

```typescript
import { animationFrameScheduler, interval } from 'rxjs';
import { map, takeWhile } from 'rxjs/operators';

@Component({
  selector: 'app-smooth-animation'
})
export class SmoothAnimationComponent implements OnInit {
  position = 0;
  target = 100;

  ngOnInit() {
    // Animate using requestAnimationFrame
    interval(0, animationFrameScheduler).pipe(
      map(() => {
        // Ease towards target
        const distance = this.target - this.position;
        return distance * 0.1;
      }),
      takeWhile(delta => Math.abs(delta) > 0.1)
    ).subscribe(delta => {
      this.position += delta;
    });
  }
}
```

### Real-World Example: Parallax Scrolling

```typescript
import { animationFrameScheduler, fromEvent } from 'rxjs';
import { map, throttleTime } from 'rxjs/operators';

@Component({
  selector: 'app-parallax'
})
export class ParallaxComponent implements OnInit {
  @ViewChild('parallaxElement') element!: ElementRef;

  ngOnInit() {
    fromEvent(window, 'scroll').pipe(
      // Throttle using animationFrame for smooth updates
      throttleTime(0, animationFrameScheduler),
      map(() => window.scrollY)
    ).subscribe(scrollY => {
      const parallaxOffset = scrollY * 0.5;
      this.element.nativeElement.style.transform = 
        `translateY(${parallaxOffset}px)`;
    });
  }
}
```

### Canvas Animation

```typescript
import { animationFrameScheduler } from 'rxjs';

@Component({
  selector: 'app-canvas-animation'
})
export class CanvasAnimationComponent implements OnInit {
  @ViewChild('canvas') canvas!: ElementRef<HTMLCanvasElement>;
  private ctx!: CanvasRenderingContext2D;
  private x = 0;

  ngOnInit() {
    this.ctx = this.canvas.nativeElement.getContext('2d')!;
    this.startAnimation();
  }

  private startAnimation() {
    const animate = () => {
      // Clear canvas
      this.ctx.clearRect(0, 0, 800, 600);

      // Draw moving circle
      this.ctx.beginPath();
      this.ctx.arc(this.x, 300, 50, 0, Math.PI * 2);
      this.ctx.fill();

      // Update position
      this.x = (this.x + 2) % 800;

      // Schedule next frame
      animationFrameScheduler.schedule(animate);
    };

    animationFrameScheduler.schedule(animate);
  }
}
```

### Performance Monitoring

```typescript
import { animationFrameScheduler } from 'rxjs';

@Injectable()
export class PerformanceMonitorService {
  private frameCount = 0;
  private lastTime = performance.now();
  private fps = 0;

  startMonitoring(): Observable<number> {
    return new Observable(observer => {
      const measure = () => {
        this.frameCount++;
        const currentTime = performance.now();
        const elapsed = currentTime - this.lastTime;

        if (elapsed >= 1000) {
          this.fps = Math.round((this.frameCount * 1000) / elapsed);
          observer.next(this.fps);
          
          this.frameCount = 0;
          this.lastTime = currentTime;
        }

        animationFrameScheduler.schedule(measure);
      };

      const subscription = animationFrameScheduler.schedule(measure);
      return () => subscription.unsubscribe();
    });
  }
}
```

## Parameterizing Schedulers

Many RxJS operators accept a scheduler parameter, allowing you to control their execution context.

### Observable Creation

```typescript
import { of, range, asyncScheduler } from 'rxjs';

// Synchronous by default
const sync$ = of(1, 2, 3);

// Make it asynchronous
const async$ = of(1, 2, 3, asyncScheduler);

// range with scheduler
const asyncRange$ = range(1, 5, asyncScheduler);
```

### Timer Operators

```typescript
import { delay, debounceTime, throttleTime, asyncScheduler } from 'rxjs';

source$.pipe(
  delay(1000, asyncScheduler),
  debounceTime(300, asyncScheduler),
  throttleTime(1000, asyncScheduler)
);
```

### Custom Scheduler Injection

```typescript
interface DataServiceConfig {
  scheduler?: SchedulerLike;
}

@Injectable()
export class ConfigurableDataService {
  private scheduler: SchedulerLike;

  constructor(@Optional() @Inject('SCHEDULER') scheduler?: SchedulerLike) {
    this.scheduler = scheduler || asyncScheduler;
  }

  getData(): Observable<Data> {
    return of(mockData, this.scheduler);
  }

  pollData(intervalMs: number): Observable<Data> {
    return interval(intervalMs, this.scheduler).pipe(
      switchMap(() => this.fetchData())
    );
  }
}
```

## Testing with Schedulers

Schedulers are essential for testing time-based observables without waiting for real time to pass.

### TestScheduler Basics

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('Observable Tests', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should delay values', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('a-b-c|');
      const expected =    '--a-b-c|';
      
      const result$ = source$.pipe(delay(20));
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

### Testing Debounce

```typescript
it('should debounce input', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const input$ = cold('a-b---c----|');
    const expected =    '------b----c|';
    
    const result$ = input$.pipe(debounceTime(30));
    
    expectObservable(result$).toBe(expected);
  });
});
```

### Testing Async Operations

```typescript
it('should handle async operations', () => {
  testScheduler.run(({ cold, hot, expectObservable, flush }) => {
    const source$ = cold('a-b-c|');
    const expected =    'a-b-c|';
    
    const result$ = source$.pipe(
      map(x => x.toUpperCase())
    );
    
    expectObservable(result$).toBe(expected, {
      a: 'A',
      b: 'B',
      c: 'C'
    });
  });
});
```

### Testing Real Service

```typescript
@Injectable()
export class TestableService {
  constructor(
    private http: HttpClient,
    @Optional() @Inject('SCHEDULER') private scheduler?: SchedulerLike
  ) {}

  pollData(): Observable<Data> {
    const scheduler = this.scheduler || asyncScheduler;
    
    return interval(1000, scheduler).pipe(
      switchMap(() => this.http.get<Data>('/api/data'))
    );
  }
}

// Test
describe('TestableService', () => {
  let service: TestableService;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    TestBed.configureTestingModule({
      providers: [
        TestableService,
        { provide: 'SCHEDULER', useValue: testScheduler }
      ]
    });

    service = TestBed.inject(TestableService);
  });

  it('should poll data every second', () => {
    testScheduler.run(({ expectObservable, cold }) => {
      const mockResponse = cold('--a|', { a: mockData });
      
      // Test polling without waiting for real time
      const result$ = service.pollData();
      
      const expected = '--a--a--a|';
      expectObservable(result$).toBe(expected);
    });
  });
});
```

## Common Mistakes

### 1. Overusing Schedulers

```typescript
// WRONG: Unnecessary scheduler usage
const data$ = of(1, 2, 3, asyncScheduler);
// Usually synchronous is fine

// RIGHT: Only use when needed
const data$ = of(1, 2, 3);
```

### 2. Wrong Scheduler for Animation

```typescript
// WRONG: Using setTimeout for animation
interval(16).pipe(
  // Uses asyncScheduler by default - not ideal for animation
)

// RIGHT: Use animationFrameScheduler
interval(0, animationFrameScheduler)
```

### 3. Not Parameterizing for Tests

```typescript
// WRONG: Hardcoded scheduler
@Injectable()
export class BadService {
  poll() {
    return interval(1000).pipe(/* ... */);
    // Can't inject test scheduler!
  }
}

// RIGHT: Injectable scheduler
@Injectable()
export class GoodService {
  constructor(@Optional() @Inject('SCHEDULER') private scheduler?: SchedulerLike) {}
  
  poll() {
    return interval(1000, this.scheduler || asyncScheduler);
  }
}
```

### 4. Memory Leaks with Schedulers

```typescript
// WRONG: Scheduled tasks never cancelled
ngOnInit() {
  asyncScheduler.schedule(() => {
    this.updateData(); // Runs even after component destroyed
  }, 1000);
}

// RIGHT: Store subscription and clean up
subscription: Subscription;

ngOnInit() {
  this.subscription = asyncScheduler.schedule(() => {
    this.updateData();
  }, 1000);
}

ngOnDestroy() {
  this.subscription?.unsubscribe();
}
```

## Best Practices

### 1. Use Default Schedulers

```typescript
// Let RxJS choose appropriate scheduler
interval(1000)
timer(1000)
delay(1000)

// Only override when you have specific needs
```

### 2. animationFrameScheduler for DOM Updates

```typescript
// Smooth, efficient DOM updates
fromEvent(window, 'scroll').pipe(
  throttleTime(0, animationFrameScheduler),
  map(() => /* calculate position */)
).subscribe(pos => {
  element.style.transform = `translateY(${pos}px)`;
});
```

### 3. Inject Schedulers for Testing

```typescript
@Injectable()
export class TestableService {
  constructor(
    @Optional() @Inject('SCHEDULER') private scheduler?: SchedulerLike
  ) {}

  // Use injected scheduler
  poll() {
    return interval(1000, this.scheduler || asyncScheduler);
  }
}
```

### 4. Document Scheduler Usage

```typescript
/**
 * Polls server for updates.
 * @param intervalMs Polling interval in milliseconds
 * @param scheduler Optional scheduler (for testing)
 */
poll(intervalMs: number, scheduler?: SchedulerLike): Observable<Data> {
  return interval(intervalMs, scheduler || asyncScheduler).pipe(
    switchMap(() => this.fetchData())
  );
}
```

## Interview Questions

### Q1: What is a scheduler in RxJS?

**Answer:** A scheduler controls when a subscription is activated and when notifications are delivered. It's like a task queue that decides the timing and execution context of observable operations. There are four main types: queueScheduler (synchronous), asapScheduler (microtask), asyncScheduler (macrotask), and animationFrameScheduler (animation frames).

### Q2: When would you use animationFrameScheduler?

**Answer:** Use `animationFrameScheduler` for smooth DOM animations and visual updates. It synchronizes operations with browser repaints (typically 60fps), preventing layout thrashing and ensuring smooth animations. Perfect for parallax effects, canvas animations, scroll animations, and any visual updates.

### Q3: How do schedulers help with testing?

**Answer:** Schedulers enable **virtual time** - tests can simulate time passing without actually waiting. Using `TestScheduler`, you can test time-based operations like delays, debouncing, and intervals instantly. You inject the test scheduler into your service, allowing you to control time in tests.

### Q4: What's the difference between asapScheduler and asyncScheduler?

**Answer:**

- `asapScheduler`: Uses microtasks (Promise.then), executes before next event loop cycle
- `asyncScheduler`: Uses macrotasks (setTimeout), executes in next event loop cycle

Microtasks have higher priority than macrotasks.

## Key Takeaways

1. Schedulers control **when** and **how** observables execute
2. **asyncScheduler** - Default for timers (setTimeout/setInterval)
3. **queueScheduler** - Synchronous, prevents stack overflow
4. **asapScheduler** - Microtask queue (Promise.then)
5. **animationFrameScheduler** - Smooth animations (requestAnimationFrame)
6. Most operators use appropriate default schedulers
7. Override schedulers for testing or specific performance needs
8. Always inject schedulers for testability
9. Use animationFrameScheduler for DOM animations
10. Virtual time testing with TestScheduler

## Resources

- [RxJS Documentation - Schedulers](https://rxjs.dev/guide/scheduler)
- [RxJS Testing Documentation](https://rxjs.dev/guide/testing/marble-testing)
- [Learn RxJS - Schedulers](https://www.learnrxjs.io/learn-rxjs/concepts/schedulers)
- [Nicholas Jamieson - RxJS Schedulers](https://ncjamieson.com/understanding-schedulers/)
