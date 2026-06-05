# Memory Management

## The Idea

**In plain English:** Memory management is how a program keeps track of the storage space it uses while running, and makes sure that space gets freed up when it is no longer needed. Think of "memory" (RAM) as temporary workspace your computer rents out to programs — good programs tidy up after themselves; bad ones keep holding onto space they are done with, causing the program to slow down or crash over time.

**Real-world analogy:** Imagine you are working at a library and you borrow a cart to hold books you are actively using. When you finish with a cart, you are supposed to return it so someone else can use it. A memory leak is like a librarian who borrows cart after cart but never returns them — eventually every cart in the library is taken and no one can get any work done.

- The library cart = a chunk of allocated memory
- Borrowing a cart = creating a variable or object in code
- Returning the cart = releasing (freeing) memory when it is no longer needed
- The library running out of carts = the app running out of memory and crashing

---

## Overview

Memory management is critical for building performant, stable web applications. Memory leaks cause applications to slow down over time, eventually crashing or becoming unresponsive. This guide covers common memory leak patterns (event listeners, closures, detached DOM nodes), garbage collection mechanics, heap profiling techniques, and prevention strategies.

## Memory Lifecycle

JavaScript uses automatic memory management through garbage collection:

```
Memory Lifecycle:
┌────────────────────────────────────────────────────────────┐
│  1. ALLOCATION    →    2. USE    →    3. RELEASE           │
├────────────────────────────────────────────────────────────┤
│  Variables created    Read/write     GC reclaims memory    │
│  Objects allocated    Computations   when unreachable      │
└────────────────────────────────────────────────────────────┘

Reachability (GC roots):
┌──────────────────────────┐
│   Global Variables       │◄─── Always reachable
├──────────────────────────┤
│   Currently Executing    │◄─── Function scope, parameters
│   Function Context       │
├──────────────────────────┤
│   Active Timers/         │◄─── setTimeout, setInterval
│   Event Listeners        │     callbacks
└──────────────────────────┘
         ↓
    Transitively reachable objects
         ↓
    (All referenced objects)
```

## Common Memory Leak Patterns

### 1. Event Listeners Not Removed

```typescript
// ❌ Bad: Event listener leak
class BadComponent {
  constructor() {
    this.data = new Array(1000000).fill('leaked data');
  }
  
  init() {
    // Adds listener but never removes it
    window.addEventListener('resize', () => {
      this.handleResize();
    });
  }
  
  handleResize() {
    console.log('Resize', this.data.length);
  }
  
  destroy() {
    // Oops! Forgot to remove listener
    // this.data is still referenced by the listener
  }
}

// Every time component is created and destroyed,
// memory leaks because listener keeps reference
const component1 = new BadComponent();
component1.init();
component1.destroy(); // Memory leaked!

const component2 = new BadComponent();
component2.init();
component2.destroy(); // More memory leaked!

// ✅ Good: Properly remove event listeners
class GoodComponent {
  private data: string[];
  private handleResizeBound: () => void;
  
  constructor() {
    this.data = new Array(1000000).fill('managed data');
    this.handleResizeBound = this.handleResize.bind(this);
  }
  
  init() {
    window.addEventListener('resize', this.handleResizeBound);
  }
  
  handleResize() {
    console.log('Resize', this.data.length);
  }
  
  destroy() {
    // Properly remove listener
    window.removeEventListener('resize', this.handleResizeBound);
    // Now this.data can be garbage collected
  }
}
```

### React Event Listener Hook (No Leaks)

```typescript
// useEventListener.ts
import { useEffect, useRef } from 'react';

export function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: Window | HTMLElement | null = window
) {
  const savedHandler = useRef(handler);
  
  // Update ref when handler changes
  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);
  
  useEffect(() => {
    if (!element?.addEventListener) return;
    
    const eventListener = (event: Event) => {
      savedHandler.current(event as WindowEventMap[K]);
    };
    
    element.addEventListener(eventName, eventListener);
    
    // Cleanup automatically removes listener
    return () => {
      element.removeEventListener(eventName, eventListener);
    };
  }, [eventName, element]);
}

// Usage - automatic cleanup!
function ResizableComponent() {
  const [width, setWidth] = useState(window.innerWidth);
  
  useEventListener('resize', () => {
    setWidth(window.innerWidth);
  });
  
  return <div>Width: {width}px</div>;
  // When component unmounts, listener is removed automatically
}
```

### Angular Event Cleanup with Subjects

```typescript
// base-component.ts
import { Component, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';

@Component({ template: '' })
export abstract class BaseComponent implements OnDestroy {
  protected destroy$ = new Subject<void>();
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// feature.component.ts
import { Component, OnInit } from '@angular/core';
import { fromEvent } from 'rxjs';
import { takeUntil, throttleTime } from 'rxjs/operators';
import { BaseComponent } from './base-component';

@Component({
  selector: 'app-feature',
  template: '<div>Width: {{ width }}</div>'
})
export class FeatureComponent extends BaseComponent implements OnInit {
  width = window.innerWidth;
  
  ngOnInit(): void {
    // Automatically unsubscribes on destroy
    fromEvent(window, 'resize')
      .pipe(
        throttleTime(100),
        takeUntil(this.destroy$)
      )
      .subscribe(() => {
        this.width = window.innerWidth;
      });
  }
  
  // ngOnDestroy inherited from BaseComponent
}
```

### 2. Closures Retaining Large Objects

```typescript
// ❌ Bad: Closure retains entire object
function createHandler() {
  const largeData = new Array(1000000).fill('data');
  
  return function handler() {
    // Only uses largeData.length, but retains entire array
    console.log(largeData.length);
  };
}

const handler = createHandler();
// largeData array is kept in memory forever!

// ✅ Good: Extract only what you need
function createHandlerOptimized() {
  const largeData = new Array(1000000).fill('data');
  const dataLength = largeData.length; // Extract primitive value
  
  // largeData can be GC'd after this function returns
  
  return function handler() {
    // Only retains the number, not the array
    console.log(dataLength);
  };
}

// ✅ Better: Explicit cleanup
function createHandlerWithCleanup() {
  let largeData: any[] | null = new Array(1000000).fill('data');
  const dataLength = largeData.length;
  
  largeData = null; // Explicit cleanup
  
  return function handler() {
    console.log(dataLength);
  };
}
```

### React Closure Pitfall

```typescript
// ❌ Bad: Closure captures large state
function ProblematicComponent() {
  const [data, setData] = useState(() => 
    new Array(1000000).fill('data')
  );
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // This interval captures 'data' in closure
    const interval = setInterval(() => {
      // Only uses count, but 'data' is captured too
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(interval);
  }, [data]); // Re-creates interval when data changes
  
  return <div>{count}</div>;
}

// ✅ Good: Avoid capturing large objects
function OptimizedComponent() {
  const [data, setData] = useState(() => 
    new Array(1000000).fill('data')
  );
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // Doesn't capture any state
    const interval = setInterval(() => {
      setCount(c => c + 1); // Functional update
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps - never re-creates
  
  return <div>{count}</div>;
}
```

### 3. Detached DOM Nodes

```typescript
// ❌ Bad: Detached DOM nodes leak
class BadDOMManager {
  private cache = new Map<string, HTMLElement>();
  
  createElement(id: string): HTMLElement {
    const element = document.createElement('div');
    element.innerHTML = '<div>'.repeat(1000); // Large DOM tree
    
    // Store in cache
    this.cache.set(id, element);
    
    return element;
  }
  
  removeElement(id: string): void {
    const element = this.cache.get(id);
    if (element?.parentNode) {
      element.parentNode.removeChild(element);
    }
    // BUG: Element still in cache!
    // DOM tree is detached but not garbage collected
  }
}

// ✅ Good: Clean up references
class GoodDOMManager {
  private cache = new Map<string, HTMLElement>();
  
  createElement(id: string): HTMLElement {
    const element = document.createElement('div');
    element.innerHTML = '<div>'.repeat(1000);
    this.cache.set(id, element);
    return element;
  }
  
  removeElement(id: string): void {
    const element = this.cache.get(id);
    if (element?.parentNode) {
      element.parentNode.removeChild(element);
    }
    
    // Remove from cache to allow GC
    this.cache.delete(id);
  }
  
  clear(): void {
    // Clear all references
    this.cache.clear();
  }
}
```

### React Detached DOM Prevention

```typescript
// useDOMCache.ts
import { useRef, useCallback, useEffect } from 'react';

export function useDOMCache() {
  const cache = useRef(new Map<string, HTMLElement>());
  
  const createElement = useCallback((id: string) => {
    const element = document.createElement('div');
    cache.current.set(id, element);
    return element;
  }, []);
  
  const removeElement = useCallback((id: string) => {
    const element = cache.current.get(id);
    if (element?.parentNode) {
      element.parentNode.removeChild(element);
    }
    cache.current.delete(id); // Clean up reference
  }, []);
  
  // Cleanup on unmount
  useEffect(() => {
    return () => {
      cache.current.clear();
    };
  }, []);
  
  return { createElement, removeElement };
}
```

### 4. Timers and Intervals Not Cleared

```typescript
// ❌ Bad: Timer leak
class BadTimer {
  private data = new Array(1000000).fill('data');
  
  start() {
    setInterval(() => {
      console.log(this.data.length);
    }, 1000);
    // Never cleared! Keeps 'this' in memory forever
  }
}

const timer = new BadTimer();
timer.start();
timer = null; // Doesn't help! Interval still running

// ✅ Good: Store and clear timer
class GoodTimer {
  private data = new Array(1000000).fill('data');
  private intervalId: NodeJS.Timeout | null = null;
  
  start() {
    this.intervalId = setInterval(() => {
      console.log(this.data.length);
    }, 1000);
  }
  
  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}
```

### React Timer Hook

```typescript
// useInterval.ts
import { useEffect, useRef } from 'react';

export function useInterval(
  callback: () => void,
  delay: number | null
) {
  const savedCallback = useRef(callback);
  
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);
  
  useEffect(() => {
    if (delay === null) return;
    
    const id = setInterval(() => {
      savedCallback.current();
    }, delay);
    
    // Automatic cleanup
    return () => clearInterval(id);
  }, [delay]);
}

// Usage
function TimerComponent() {
  const [count, setCount] = useState(0);
  
  useInterval(() => {
    setCount(c => c + 1);
  }, 1000);
  
  return <div>{count}</div>;
  // Timer automatically cleaned up on unmount
}
```

### 5. Global Variables and Window Properties

```typescript
// ❌ Bad: Polluting global scope
window.myAppData = {
  users: new Array(100000).fill({ name: 'User', data: {} }),
  cache: new Map(),
  // This stays in memory forever!
};

function processData() {
  window.myAppData.users.forEach(user => {
    // Process users
  });
}

// ✅ Good: Use modules and proper scoping
class DataManager {
  private users: User[];
  private cache: Map<string, any>;
  
  constructor() {
    this.users = [];
    this.cache = new Map();
  }
  
  cleanup() {
    this.users = [];
    this.cache.clear();
  }
}

const dataManager = new DataManager();
// Can be garbage collected when no longer referenced

export { dataManager };
```

## Garbage Collection

JavaScript uses mark-and-sweep garbage collection:

```
Mark-and-Sweep Algorithm:
┌──────────────────────────────────────────────────────────┐
│ Phase 1: MARK                                            │
│                                                          │
│  GC Roots (global, stack) → Mark reachable objects      │
│         ↓                                                │
│    Follow references                                     │
│         ↓                                                │
│    Mark all transitively reachable objects              │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│ Phase 2: SWEEP                                           │
│                                                          │
│  Scan entire heap                                        │
│         ↓                                                │
│  Unmarked objects = unreachable = garbage                │
│         ↓                                                │
│  Reclaim memory                                          │
└──────────────────────────────────────────────────────────┘

Example:
┌─────────────┐
│   Global    │
└──────┬──────┘
       │ ✓ (marked)
       ↓
   ┌───────┐     ┌─────────┐
   │ Obj A │────→│  Obj B  │ ✓ (marked, reachable)
   └───────┘     └─────────┘
   
   ┌─────────┐
   │  Obj C  │ ✗ (unmarked, unreachable, GC'd)
   └─────────┘
```

### Generational Garbage Collection

V8 uses generational GC with two heaps:

```
Heap Structure:
┌────────────────────────────────────────────────────────┐
│                   Young Generation                     │
│              (New Space: 1-8 MB)                       │
├────────────────────────────────────────────────────────┤
│  - Fast allocation                                     │
│  - Frequent, fast GC (Scavenge)                       │
│  - Most objects die young                              │
│  - Surviving objects promoted to Old Generation        │
└────────────────────────────────────────────────────────┘
                        ↓ (promotion)
┌────────────────────────────────────────────────────────┐
│                   Old Generation                       │
│              (Old Space: larger)                       │
├────────────────────────────────────────────────────────┤
│  - Long-lived objects                                  │
│  - Infrequent, slower GC (Mark-Sweep-Compact)         │
│  - Larger space                                        │
└────────────────────────────────────────────────────────┘
```

### Monitoring GC

```typescript
// Monitor GC events (Chrome)
if ('PerformanceObserver' in window) {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      console.log('GC event:', {
        type: entry.name,
        duration: entry.duration,
        startTime: entry.startTime
      });
    }
  });
  
  observer.observe({ entryTypes: ['measure'] });
}

// Force GC in Node.js (with --expose-gc flag)
if (global.gc) {
  global.gc();
  const used = process.memoryUsage();
  console.log('Memory after GC:', {
    heapUsed: (used.heapUsed / 1024 / 1024).toFixed(2) + ' MB',
    heapTotal: (used.heapTotal / 1024 / 1024).toFixed(2) + ' MB'
  });
}
```

## Memory Profiling

### Chrome DevTools Memory Profiler

```typescript
// 1. Heap Snapshot
// Take snapshots before/after operations to find leaks

// Example: Detect leak
class LeakyComponent {
  private data: any[] = [];
  
  init() {
    // Take snapshot 1 here
    
    for (let i = 0; i < 1000; i++) {
      this.addItem();
    }
    
    // Take snapshot 2 here
    
    this.cleanup();
    
    // Take snapshot 3 here
    // Compare: If snapshot 3 > snapshot 1, there's a leak
  }
  
  addItem() {
    this.data.push(new Array(10000).fill('data'));
  }
  
  cleanup() {
    // Supposed to clean up, but doesn't
    // this.data = []; // Forgot to clear!
  }
}

// 2. Allocation Timeline
// Records allocations over time to find memory leaks

// 3. Allocation Sampling
// Less overhead, good for production profiling
```

### React DevTools Profiler

```typescript
// Profiler.tsx
import { Profiler, ProfilerOnRenderCallback } from 'react';

const onRenderCallback: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime,
  interactions
) => {
  console.log('Render profiling:', {
    id,
    phase, // "mount" or "update"
    actualDuration, // Time spent rendering
    baseDuration, // Estimated time without memoization
    startTime,
    commitTime
  });
  
  // Send to analytics
  if (actualDuration > 100) {
    analytics.track('slow_render', {
      component: id,
      duration: actualDuration
    });
  }
};

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <ExpensiveComponent />
    </Profiler>
  );
}
```

### Memory Monitoring Hook

```typescript
// useMemoryMonitor.ts
import { useEffect, useState } from 'react';

interface MemoryInfo {
  usedJSHeapSize: number;
  totalJSHeapSize: number;
  jsHeapSizeLimit: number;
}

export function useMemoryMonitor(interval = 1000) {
  const [memory, setMemory] = useState<MemoryInfo | null>(null);
  
  useEffect(() => {
    if (!('memory' in performance)) {
      console.warn('Memory API not available');
      return;
    }
    
    const updateMemory = () => {
      const memInfo = (performance as any).memory;
      setMemory({
        usedJSHeapSize: memInfo.usedJSHeapSize,
        totalJSHeapSize: memInfo.totalJSHeapSize,
        jsHeapSizeLimit: memInfo.jsHeapSizeLimit
      });
    };
    
    updateMemory();
    const id = setInterval(updateMemory, interval);
    
    return () => clearInterval(id);
  }, [interval]);
  
  return memory;
}

// Usage
function MemoryMonitor() {
  const memory = useMemoryMonitor(1000);
  
  if (!memory) return <div>Memory API not available</div>;
  
  const usedMB = (memory.usedJSHeapSize / 1024 / 1024).toFixed(2);
  const limitMB = (memory.jsHeapSizeLimit / 1024 / 1024).toFixed(2);
  const percentage = (
    (memory.usedJSHeapSize / memory.jsHeapSizeLimit) * 100
  ).toFixed(1);
  
  return (
    <div>
      <div>Memory: {usedMB} MB / {limitMB} MB</div>
      <div>Usage: {percentage}%</div>
      {parseFloat(percentage) > 90 && (
        <div style={{ color: 'red' }}>
          Warning: High memory usage!
        </div>
      )}
    </div>
  );
}
```

## WeakMap and WeakSet

WeakMap and WeakSet don't prevent garbage collection:

```typescript
// Regular Map prevents GC
const regularMap = new Map();
let obj = { data: new Array(1000000).fill('data') };
regularMap.set(obj, 'metadata');
obj = null; // Object is NOT garbage collected (Map still references it)

// WeakMap allows GC
const weakMap = new WeakMap();
let obj2 = { data: new Array(1000000).fill('data') };
weakMap.set(obj2, 'metadata');
obj2 = null; // Object CAN be garbage collected!

// Use cases for WeakMap
class DOMNodeMetadata {
  private metadata = new WeakMap<HTMLElement, any>();
  
  setMetadata(element: HTMLElement, data: any) {
    this.metadata.set(element, data);
    // When element is removed from DOM and has no other references,
    // it can be GC'd along with its metadata
  }
  
  getMetadata(element: HTMLElement) {
    return this.metadata.get(element);
  }
}

// React example with WeakMap
const componentCache = new WeakMap<object, React.ComponentType>();

function getCachedComponent(key: object): React.ComponentType | undefined {
  return componentCache.get(key);
}

function setCachedComponent(key: object, component: React.ComponentType) {
  componentCache.set(key, component);
  // When key object is GC'd, cached component is too
}
```

### Angular WeakMap for Metadata

```typescript
// metadata.service.ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class MetadataService {
  private metadata = new WeakMap<object, any>();
  
  set(target: object, data: any): void {
    this.metadata.set(target, data);
  }
  
  get(target: object): any {
    return this.metadata.get(target);
  }
  
  has(target: object): boolean {
    return this.metadata.has(target);
  }
  
  // No need for cleanup - WeakMap handles it automatically
}

// Usage
@Component({
  selector: 'app-user-list',
  template: '<div *ngFor="let user of users">{{ user.name }}</div>'
})
export class UserListComponent {
  users: User[] = [];
  
  constructor(private metadata: MetadataService) {}
  
  ngOnInit(): void {
    this.users.forEach(user => {
      // Store metadata without preventing GC
      this.metadata.set(user, {
        loadTime: Date.now(),
        viewCount: 0
      });
    });
  }
  
  // When users array is replaced, old user objects
  // and their metadata can be garbage collected
}
```

## Memory Leak Detection

### Automated Leak Detection

```typescript
// MemoryLeakDetector.ts
export class MemoryLeakDetector {
  private snapshots: number[] = [];
  private threshold: number;
  
  constructor(thresholdMB = 10) {
    this.threshold = thresholdMB * 1024 * 1024;
  }
  
  takeSnapshot(): void {
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      this.snapshots.push(memory.usedJSHeapSize);
      
      // Keep only last 10 snapshots
      if (this.snapshots.length > 10) {
        this.snapshots.shift();
      }
      
      this.analyze();
    }
  }
  
  private analyze(): void {
    if (this.snapshots.length < 5) return;
    
    const first = this.snapshots[0];
    const last = this.snapshots[this.snapshots.length - 1];
    const growth = last - first;
    
    // Check for consistent memory growth
    const isGrowing = this.snapshots.every((val, i) => 
      i === 0 || val >= this.snapshots[i - 1]
    );
    
    if (isGrowing && growth > this.threshold) {
      console.warn('Potential memory leak detected!', {
        growth: (growth / 1024 / 1024).toFixed(2) + ' MB',
        snapshots: this.snapshots.map(s => 
          (s / 1024 / 1024).toFixed(2) + ' MB'
        )
      });
      
      // Send alert
      this.reportLeak(growth);
    }
  }
  
  private reportLeak(growth: number): void {
    // Send to monitoring service
    if (typeof analytics !== 'undefined') {
      analytics.track('memory_leak_detected', {
        growth: growth / 1024 / 1024,
        url: window.location.href
      });
    }
  }
  
  startMonitoring(intervalMs = 30000): () => void {
    const id = setInterval(() => {
      this.takeSnapshot();
    }, intervalMs);
    
    return () => clearInterval(id);
  }
}

// Usage
const detector = new MemoryLeakDetector(10); // 10MB threshold
const stopMonitoring = detector.startMonitoring(30000); // Check every 30s

// Later: stopMonitoring();
```

### React Leak Detection Component

```typescript
// LeakDetector.tsx
import React, { useEffect, useRef } from 'react';

export const LeakDetector: React.FC = () => {
  const snapshots = useRef<number[]>([]);
  
  useEffect(() => {
    const checkMemory = () => {
      if (!('memory' in performance)) return;
      
      const memory = (performance as any).memory;
      const used = memory.usedJSHeapSize;
      
      snapshots.current.push(used);
      
      if (snapshots.current.length > 10) {
        snapshots.current.shift();
      }
      
      // Detect growth trend
      if (snapshots.current.length >= 5) {
        const growth = used - snapshots.current[0];
        const growthMB = growth / 1024 / 1024;
        
        if (growthMB > 10) {
          console.warn('Memory growing:', growthMB.toFixed(2), 'MB');
        }
      }
    };
    
    const interval = setInterval(checkMemory, 30000);
    
    return () => clearInterval(interval);
  }, []);
  
  return null; // No UI
};

// Usage in App
function App() {
  return (
    <>
      {process.env.NODE_ENV === 'development' && <LeakDetector />}
      <YourApp />
    </>
  );
}
```

## Common Mistakes

### 1. Not Removing Event Listeners

```typescript
// ❌ Bad
element.addEventListener('click', handleClick);
// Never removed!

// ✅ Good
element.addEventListener('click', handleClick);
// Later:
element.removeEventListener('click', handleClick);
```

### 2. Storing DOM References in Data Structures

```typescript
// ❌ Bad
const cache = new Map<string, HTMLElement>();
cache.set('node', document.getElementById('myDiv')!);
// When element is removed from DOM, it can't be GC'd

// ✅ Good: Use WeakMap
const cache = new WeakMap<HTMLElement, any>();
const element = document.getElementById('myDiv')!;
cache.set(element, data);
// When element is removed from DOM, it can be GC'd
```

### 3. Circular References

```typescript
// ❌ Bad: Circular reference
const obj1: any = {};
const obj2: any = {};
obj1.ref = obj2;
obj2.ref = obj1;
// Modern GC handles this, but avoid when possible

// ✅ Good: Break cycles manually
obj1.ref = null;
obj2.ref = null;
```

### 4. Not Clearing Timers

```typescript
// ❌ Bad
setInterval(() => {
  console.log(bigData.length);
}, 1000);
// Never cleared, keeps bigData in memory

// ✅ Good
const id = setInterval(() => {
  console.log(bigData.length);
}, 1000);
// Later:
clearInterval(id);
```

### 5. Large Objects in Closures

```typescript
// ❌ Bad
function createHandler() {
  const hugeArray = new Array(1000000);
  return () => console.log(hugeArray.length);
}

// ✅ Good
function createHandler() {
  const hugeArray = new Array(1000000);
  const length = hugeArray.length;
  return () => console.log(length);
}
```

## Best Practices

1. **Always remove event listeners** when components unmount
2. **Use WeakMap/WeakSet** for caching with DOM nodes or objects
3. **Clear timers and intervals** explicitly
4. **Avoid storing large objects in closures** - extract only what you need
5. **Profile regularly** with Chrome DevTools Memory Profiler
6. **Monitor memory in production** with performance.memory API
7. **Use React hooks correctly** - proper cleanup in useEffect
8. **Nullify references** to large objects when done
9. **Avoid global variables** - use proper scoping and modules
10. **Test for leaks** - create/destroy components repeatedly and check memory

## When to Profile

### Profile when:
- Page feels slower over time
- Memory warnings in DevTools
- High memory usage in production
- Before major refactoring
- After adding new features

### Don't profile:
- Prematurely (profile after identifying issues)
- In production without sampling
- Without specific goals

## Interview Questions

### 1. What causes memory leaks in JavaScript applications?

**Answer**: Common causes include: (1) Event listeners not removed on cleanup, (2) Timers/intervals not cleared, (3) Closures retaining large objects, (4) Detached DOM nodes stored in data structures, (5) Global variables never nullified, (6) Circular references (rare in modern GC). The key is that objects remain reachable from GC roots even though they're no longer needed by the application.

### 2. Explain the difference between Map and WeakMap.

**Answer**: Map maintains strong references preventing garbage collection - if an object is a Map key, it won't be GC'd even if no other references exist. WeakMap uses weak references - keys can be garbage collected when no other references exist. WeakMap keys must be objects (not primitives), has no size property, and isn't iterable. Use WeakMap for metadata on objects/DOM nodes where you don't want to prevent GC.

### 3. How does garbage collection work in V8?

**Answer**: V8 uses generational garbage collection with two heaps: Young Generation (1-8MB) for short-lived objects with frequent, fast GC (Scavenge algorithm), and Old Generation for long-lived objects with infrequent, slower GC (Mark-Sweep-Compact). Objects start in Young Generation and are promoted to Old Generation after surviving multiple GC cycles. This exploits the "generational hypothesis" that most objects die young.

### 4. What are detached DOM nodes and why are they problematic?

**Answer**: Detached DOM nodes are nodes removed from the DOM tree but still referenced in JavaScript (e.g., in arrays, maps, closures). They're problematic because they consume memory (including all child nodes) but serve no purpose since they're not visible. They prevent garbage collection until all JavaScript references are removed. Detect them in Chrome DevTools Memory Profiler by searching for "Detached" in heap snapshots.

### 5. How would you detect a memory leak in a React application?

**Answer**: 
1. Use React DevTools Profiler to identify problematic components
2. Take heap snapshots before/after component mount/unmount cycles
3. Monitor memory growth with performance.memory API
4. Check for event listeners not cleaned up in useEffect
5. Look for timers/intervals without cleanup
6. Profile with Chrome DevTools Memory → Allocation Timeline
7. Use automated tools like fuite or memlab

### 6. Explain why this code leaks memory:

```typescript
class Component {
  init() {
    window.addEventListener('scroll', () => {
      this.handleScroll();
    });
  }
}
```

**Answer**: The arrow function closure captures `this`, maintaining a reference to the Component instance. When the component is "destroyed," the event listener remains attached to window, keeping the Component in memory. The listener must be explicitly removed using removeEventListener. Solution: Store bound function reference and remove it in cleanup: `window.removeEventListener('scroll', this.handleScrollBound)`.

### 7. What tools would you use to profile memory in production?

**Answer**: 
1. performance.memory API for basic heap size monitoring
2. PerformanceObserver for long tasks that might cause memory issues
3. Sampling heap profiler with lower overhead than full profiling
4. Memory leak detection libraries (automated snapshots)
5. RUM (Real User Monitoring) with memory metrics
6. Error tracking with memory context (heap size when errors occur)
7. Regular automated heap snapshot comparisons in staging

### 8. How do React hooks help prevent memory leaks?

**Answer**: Hooks enforce cleanup patterns through useEffect return functions. When a component unmounts, all useEffect cleanup functions run automatically, providing a clear place to remove listeners, clear timers, and cancel requests. Custom hooks encapsulate cleanup logic, making it reusable. useCallback/useMemo prevent unnecessary closure recreation. However, hooks don't automatically prevent leaks - developers must still implement proper cleanup in useEffect returns.

## Key Takeaways

1. **Always clean up event listeners** - most common source of memory leaks
2. **Use WeakMap for object/DOM metadata** - allows garbage collection
3. **Clear all timers and intervals** - setInterval/setTimeout must be cleared
4. **Profile memory regularly** - use Chrome DevTools heap snapshots
5. **Monitor detached DOM nodes** - remove from caches when deleted from DOM
6. **Minimize closure scope** - extract primitives instead of capturing large objects
7. **Use React hooks cleanup** - return cleanup functions from useEffect
8. **Avoid global variables** - prefer modules and proper scoping
9. **Test component lifecycle** - mount/unmount repeatedly to detect leaks
10. **Monitor production memory** - use performance.memory API for early detection

## Resources

- [Memory Management (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [Chrome DevTools Memory Profiler](https://developer.chrome.com/docs/devtools/memory-problems/)
- [Debugging Memory Leaks (web.dev)](https://web.dev/articles/debugging-memory-leaks)
- [WeakMap and WeakSet (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
- [V8 Garbage Collection](https://v8.dev/blog/trash-talk)
- [Finding Memory Leaks in React](https://kentcdodds.com/blog/fix-the-slow-render-before-you-fix-the-re-render)
- [fuite - Memory Leak Testing](https://github.com/nolanlawson/fuite)
