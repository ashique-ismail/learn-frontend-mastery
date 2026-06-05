# Runtime Performance

## The Idea

**In plain English:** Runtime performance is about how smoothly a webpage responds while you are using it -- things like clicking buttons, scrolling, and typing. When JavaScript (the code that makes pages interactive) takes too long to finish a task, the browser gets stuck and cannot react to what you do, making the page feel frozen or laggy.

**Real-world analogy:** Imagine a single cashier at a grocery store who also has to stock shelves, answer the phone, and handle complaints -- all by themselves. If a customer asks them to count every item in the stockroom (a huge task), they disappear for ten minutes and the checkout line piles up with frustrated customers.

- The cashier = the browser's main thread (the one worker handling everything)
- Counting every item in the stockroom = a long-running JavaScript task
- Customers waiting in line = user clicks and interactions that cannot be handled
- Breaking the stockroom count into small batches between customers = chunking tasks so the browser stays responsive

---

## Overview

Runtime performance refers to how efficiently JavaScript executes in the browser after the page has loaded. Poor runtime performance causes sluggish interactions, janky scrolling, unresponsive UIs, and frozen pages. This guide covers JavaScript execution optimization, long task management, main thread blocking prevention, requestIdleCallback, and Web Workers for parallel computation.

## The Main Thread

The browser's main thread handles JavaScript execution, rendering, layout, and user interactions:

```
Main Thread Execution Model:
┌──────────────────────────────────────────────────────────────┐
│                       Main Thread                             │
├──────────────────────────────────────────────────────────────┤
│  JS Execution → Style Calc → Layout → Paint → Composite      │
│     ↓              ↓            ↓        ↓          ↓         │
│  Blocking      Blocking     Blocking  Blocking  Blocking     │
└──────────────────────────────────────────────────────────────┘

Long Task (>50ms):
┌──────────────────────────────────────────────────────────────┐
│  [===== JavaScript Execution (300ms) =====]                  │
│                                              ↑                │
│                                        User clicks here       │
│                                        (no response!)         │
└──────────────────────────────────────────────────────────────┘

Optimized (chunked):
┌──────────────────────────────────────────────────────────────┐
│  [== JS ==] [render] [== JS ==] [render] [== JS ==] [render] │
│     (30ms)   (16ms)    (30ms)    (16ms)    (30ms)   (16ms)   │
│                         ↑                                     │
│                    User clicks (responds immediately!)        │
└──────────────────────────────────────────────────────────────┘
```

### Main Thread Blocking Example

```typescript
// ❌ Bad: Blocks main thread for 2+ seconds
function processLargeDataset(data: any[]) {
  const results = [];
  
  // Synchronous loop blocks everything
  for (let i = 0; i < 1000000; i++) {
    results.push(expensiveCalculation(data[i]));
  }
  
  return results;
}

function expensiveCalculation(item: any) {
  // Complex computation
  let result = 0;
  for (let i = 0; i < 10000; i++) {
    result += Math.sqrt(i) * Math.random();
  }
  return result;
}

// This blocks the UI completely
const results = processLargeDataset(hugeArray);
// User can't click, scroll, or interact during this time
```

## Long Tasks

Tasks longer than 50ms are considered "long tasks" that block user interactions:

### Detecting Long Tasks

```typescript
// Long Task Observer
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long task detected:', {
      duration: entry.duration,
      startTime: entry.startTime,
      attribution: (entry as any).attribution
    });
    
    // Send to analytics
    if (entry.duration > 100) {
      analytics.track('long_task', {
        duration: entry.duration,
        url: window.location.href
      });
    }
  }
});

observer.observe({ entryTypes: ['longtask'] });
```

### React Long Task Monitor

```typescript
// LongTaskMonitor.tsx
import { useEffect, useRef } from 'react';

interface LongTaskEntry {
  duration: number;
  startTime: number;
  timestamp: number;
}

export function useLongTaskMonitor(
  threshold = 50,
  onLongTask?: (entry: LongTaskEntry) => void
) {
  const longTasks = useRef<LongTaskEntry[]>([]);
  
  useEffect(() => {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration >= threshold) {
          const taskEntry: LongTaskEntry = {
            duration: entry.duration,
            startTime: entry.startTime,
            timestamp: Date.now()
          };
          
          longTasks.current.push(taskEntry);
          onLongTask?.(taskEntry);
        }
      }
    });
    
    observer.observe({ entryTypes: ['longtask'] });
    
    return () => observer.disconnect();
  }, [threshold, onLongTask]);
  
  return longTasks;
}

// Usage
function App() {
  useLongTaskMonitor(50, (entry) => {
    console.warn(`Long task: ${entry.duration.toFixed(2)}ms`);
  });
  
  return <div>App content</div>;
}
```

### Angular Long Task Service

```typescript
// long-task.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Subject, Observable } from 'rxjs';

export interface LongTask {
  duration: number;
  startTime: number;
  timestamp: Date;
}

@Injectable({
  providedIn: 'root'
})
export class LongTaskService implements OnDestroy {
  private longTasks$ = new Subject<LongTask>();
  private observer: PerformanceObserver | null = null;
  
  constructor() {
    this.initObserver();
  }
  
  private initObserver(): void {
    if ('PerformanceObserver' in window) {
      this.observer = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          this.longTasks$.next({
            duration: entry.duration,
            startTime: entry.startTime,
            timestamp: new Date()
          });
        }
      });
      
      this.observer.observe({ entryTypes: ['longtask'] });
    }
  }
  
  getLongTasks(): Observable<LongTask> {
    return this.longTasks$.asObservable();
  }
  
  ngOnDestroy(): void {
    this.observer?.disconnect();
    this.longTasks$.complete();
  }
}

// Usage in component
@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent implements OnInit {
  constructor(private longTaskService: LongTaskService) {}
  
  ngOnInit(): void {
    this.longTaskService.getLongTasks().subscribe(task => {
      if (task.duration > 100) {
        console.warn('Critical long task:', task);
      }
    });
  }
}
```

## Breaking Up Long Tasks

### Task Chunking with setTimeout

```typescript
// ✅ Good: Chunk work into smaller tasks
async function processLargeDatasetChunked(data: any[]) {
  const results = [];
  const chunkSize = 100;
  
  for (let i = 0; i < data.length; i += chunkSize) {
    const chunk = data.slice(i, i + chunkSize);
    
    // Process chunk
    for (const item of chunk) {
      results.push(expensiveCalculation(item));
    }
    
    // Yield to browser to handle user interactions
    if (i + chunkSize < data.length) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
  
  return results;
}

// Usage
async function handleDataProcessing() {
  showLoadingSpinner();
  
  try {
    const results = await processLargeDatasetChunked(hugeArray);
    updateUI(results);
  } finally {
    hideLoadingSpinner();
  }
}
```

### React Chunked Processing Hook

```typescript
// useChunkedProcessing.ts
import { useState, useCallback, useRef } from 'react';

interface ChunkedProcessingOptions<T, R> {
  data: T[];
  processor: (item: T) => R;
  chunkSize?: number;
  onProgress?: (processed: number, total: number) => void;
  onComplete?: (results: R[]) => void;
}

export function useChunkedProcessing<T, R>() {
  const [processing, setProcessing] = useState(false);
  const [progress, setProgress] = useState(0);
  const cancelRef = useRef(false);
  
  const process = useCallback(async ({
    data,
    processor,
    chunkSize = 100,
    onProgress,
    onComplete
  }: ChunkedProcessingOptions<T, R>) => {
    setProcessing(true);
    setProgress(0);
    cancelRef.current = false;
    
    const results: R[] = [];
    
    try {
      for (let i = 0; i < data.length; i += chunkSize) {
        if (cancelRef.current) break;
        
        const chunk = data.slice(i, i + chunkSize);
        
        // Process chunk
        for (const item of chunk) {
          results.push(processor(item));
        }
        
        // Update progress
        const processed = Math.min(i + chunkSize, data.length);
        setProgress(processed / data.length);
        onProgress?.(processed, data.length);
        
        // Yield to browser
        await new Promise(resolve => setTimeout(resolve, 0));
      }
      
      if (!cancelRef.current) {
        onComplete?.(results);
      }
    } finally {
      setProcessing(false);
    }
    
    return results;
  }, []);
  
  const cancel = useCallback(() => {
    cancelRef.current = true;
  }, []);
  
  return { process, cancel, processing, progress };
}

// Usage
function DataProcessor() {
  const { process, cancel, processing, progress } = useChunkedProcessing<
    number,
    number
  >();
  const [results, setResults] = useState<number[]>([]);
  
  const handleProcess = async () => {
    const data = Array.from({ length: 10000 }, (_, i) => i);
    
    await process({
      data,
      processor: (num) => num * num,
      chunkSize: 100,
      onProgress: (processed, total) => {
        console.log(`Progress: ${processed}/${total}`);
      },
      onComplete: (results) => {
        setResults(results);
        console.log('Processing complete!');
      }
    });
  };
  
  return (
    <div>
      <button onClick={handleProcess} disabled={processing}>
        Process Data
      </button>
      {processing && (
        <>
          <div>Progress: {(progress * 100).toFixed(0)}%</div>
          <button onClick={cancel}>Cancel</button>
        </>
      )}
      <div>Results: {results.length} items</div>
    </div>
  );
}
```

## requestIdleCallback

Execute non-critical work during browser idle time:

```typescript
// Basic requestIdleCallback usage
function performNonCriticalWork() {
  requestIdleCallback((deadline) => {
    // deadline.timeRemaining() returns available time in ms
    while (deadline.timeRemaining() > 0 && workQueue.length > 0) {
      const work = workQueue.shift();
      work();
    }
    
    // If more work remains, schedule another callback
    if (workQueue.length > 0) {
      performNonCriticalWork();
    }
  }, { timeout: 2000 }); // Force execution after 2s if still no idle time
}

// Work queue example
const workQueue: Array<() => void> = [];

function scheduleWork(task: () => void) {
  workQueue.push(task);
  
  if (workQueue.length === 1) {
    performNonCriticalWork();
  }
}

// Schedule non-critical analytics
scheduleWork(() => {
  analytics.track('page_view', { url: window.location.href });
});

// Schedule lazy loading
scheduleWork(() => {
  preloadNextPageImages();
});
```

### React Idle Callback Hook

```typescript
// useIdleCallback.ts
import { useEffect, useRef, useCallback } from 'react';

export function useIdleCallback(
  callback: () => void,
  options?: IdleRequestOptions
) {
  const callbackRef = useRef(callback);
  const handleRef = useRef<number>();
  
  // Update callback ref
  useEffect(() => {
    callbackRef.current = callback;
  }, [callback]);
  
  const schedule = useCallback(() => {
    if (handleRef.current) {
      cancelIdleCallback(handleRef.current);
    }
    
    handleRef.current = requestIdleCallback(() => {
      callbackRef.current();
    }, options);
  }, [options]);
  
  const cancel = useCallback(() => {
    if (handleRef.current) {
      cancelIdleCallback(handleRef.current);
      handleRef.current = undefined;
    }
  }, []);
  
  useEffect(() => {
    return cancel;
  }, [cancel]);
  
  return { schedule, cancel };
}

// Usage
function AnalyticsTracker() {
  const { schedule } = useIdleCallback(() => {
    // Send analytics during idle time
    analytics.flush();
  }, { timeout: 2000 });
  
  const trackEvent = (event: string) => {
    analytics.queue(event);
    schedule(); // Schedule flush during idle time
  };
  
  return (
    <button onClick={() => trackEvent('button_click')}>
      Track Event
    </button>
  );
}
```

### Angular Idle Task Service

```typescript
// idle-task.service.ts
import { Injectable, NgZone, OnDestroy } from '@angular/core';

type Task = () => void;

@Injectable({
  providedIn: 'root'
})
export class IdleTaskService implements OnDestroy {
  private taskQueue: Task[] = [];
  private isProcessing = false;
  private handleId?: number;
  
  constructor(private zone: NgZone) {}
  
  scheduleTask(task: Task, options?: IdleRequestOptions): void {
    this.taskQueue.push(task);
    
    if (!this.isProcessing) {
      this.processQueue(options);
    }
  }
  
  private processQueue(options?: IdleRequestOptions): void {
    this.isProcessing = true;
    
    // Run outside Angular zone to avoid triggering change detection
    this.zone.runOutsideAngular(() => {
      this.handleId = requestIdleCallback((deadline) => {
        while (deadline.timeRemaining() > 0 && this.taskQueue.length > 0) {
          const task = this.taskQueue.shift();
          task?.();
        }
        
        if (this.taskQueue.length > 0) {
          this.processQueue(options);
        } else {
          this.isProcessing = false;
        }
      }, options);
    });
  }
  
  cancelAll(): void {
    if (this.handleId) {
      cancelIdleCallback(this.handleId);
      this.handleId = undefined;
    }
    this.taskQueue = [];
    this.isProcessing = false;
  }
  
  ngOnDestroy(): void {
    this.cancelAll();
  }
}

// Usage
@Component({
  selector: 'app-dashboard',
  template: '<div>Dashboard</div>'
})
export class DashboardComponent implements OnInit {
  constructor(private idleTask: IdleTaskService) {}
  
  ngOnInit(): void {
    // Schedule non-critical work
    this.idleTask.scheduleTask(() => {
      this.preloadReports();
    });
    
    this.idleTask.scheduleTask(() => {
      this.updateCache();
    }, { timeout: 3000 });
  }
  
  preloadReports(): void {
    // Preload data for better UX
  }
  
  updateCache(): void {
    // Update local cache
  }
}
```

## Web Workers

Web Workers run JavaScript in background threads, off the main thread:

```
Architecture:
┌─────────────────────┐         ┌─────────────────────┐
│   Main Thread       │         │   Worker Thread     │
│                     │         │                     │
│  - UI Updates       │◄───────►│  - Heavy Compute    │
│  - User Events      │ postMsg │  - Data Processing  │
│  - Rendering        │         │  - Parsing          │
└─────────────────────┘         └─────────────────────┘
      Non-blocking                   Background work
```

### Basic Web Worker

```typescript
// worker.ts
self.addEventListener('message', (event) => {
  const { type, data } = event.data;
  
  switch (type) {
    case 'PROCESS_DATA':
      const result = processData(data);
      self.postMessage({ type: 'RESULT', data: result });
      break;
      
    case 'HEAVY_CALCULATION':
      const calculated = heavyCalculation(data);
      self.postMessage({ type: 'CALCULATED', data: calculated });
      break;
  }
});

function processData(data: any[]) {
  return data.map(item => {
    // Expensive operation
    return expensiveTransform(item);
  });
}

function heavyCalculation(input: number): number {
  let result = 0;
  for (let i = 0; i < 10000000; i++) {
    result += Math.sqrt(i) * input;
  }
  return result;
}

// main.ts
const worker = new Worker(new URL('./worker.ts', import.meta.url), {
  type: 'module'
});

worker.addEventListener('message', (event) => {
  const { type, data } = event.data;
  
  if (type === 'RESULT') {
    console.log('Processed data:', data);
    updateUI(data);
  }
});

worker.addEventListener('error', (error) => {
  console.error('Worker error:', error);
});

// Send work to worker (doesn't block main thread!)
worker.postMessage({
  type: 'PROCESS_DATA',
  data: largeDataset
});
```

### React Web Worker Hook

```typescript
// useWebWorker.ts
import { useEffect, useRef, useCallback, useState } from 'react';

interface WorkerMessage<T = any> {
  type: string;
  data: T;
}

export function useWebWorker<T = any, R = any>(
  workerUrl: string
) {
  const workerRef = useRef<Worker>();
  const [result, setResult] = useState<R | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    workerRef.current = new Worker(workerUrl, { type: 'module' });
    
    workerRef.current.addEventListener('message', (event: MessageEvent<WorkerMessage<R>>) => {
      setResult(event.data.data);
      setLoading(false);
    });
    
    workerRef.current.addEventListener('error', (error) => {
      setError(error as any);
      setLoading(false);
    });
    
    return () => {
      workerRef.current?.terminate();
    };
  }, [workerUrl]);
  
  const postMessage = useCallback((type: string, data: T) => {
    if (!workerRef.current) {
      setError(new Error('Worker not initialized'));
      return;
    }
    
    setLoading(true);
    setError(null);
    workerRef.current.postMessage({ type, data });
  }, []);
  
  return { postMessage, result, error, loading };
}

// Usage
function HeavyComputation() {
  const { postMessage, result, error, loading } = useWebWorker<number[], number[]>(
    new URL('./computation.worker.ts', import.meta.url).toString()
  );
  
  const handleCompute = () => {
    const data = Array.from({ length: 100000 }, (_, i) => i);
    postMessage('COMPUTE', data);
  };
  
  return (
    <div>
      <button onClick={handleCompute} disabled={loading}>
        {loading ? 'Computing...' : 'Compute'}
      </button>
      {error && <div>Error: {error.message}</div>}
      {result && <div>Result: {result.length} items processed</div>}
    </div>
  );
}
```

### Advanced Worker Pool

```typescript
// WorkerPool.ts
export class WorkerPool {
  private workers: Worker[] = [];
  private availableWorkers: Worker[] = [];
  private taskQueue: Array<{
    data: any;
    resolve: (value: any) => void;
    reject: (error: any) => void;
  }> = [];
  
  constructor(workerUrl: string, poolSize: number = navigator.hardwareConcurrency || 4) {
    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerUrl, { type: 'module' });
      
      worker.addEventListener('message', (event) => {
        this.handleWorkerMessage(worker, event.data);
      });
      
      worker.addEventListener('error', (error) => {
        this.handleWorkerError(worker, error);
      });
      
      this.workers.push(worker);
      this.availableWorkers.push(worker);
    }
  }
  
  async execute<T, R>(type: string, data: T): Promise<R> {
    return new Promise((resolve, reject) => {
      const task = { data: { type, data }, resolve, reject };
      
      if (this.availableWorkers.length > 0) {
        this.runTask(task);
      } else {
        this.taskQueue.push(task);
      }
    });
  }
  
  private runTask(task: any): void {
    const worker = this.availableWorkers.pop()!;
    
    // Store task info on worker for later
    (worker as any).__currentTask = task;
    
    worker.postMessage(task.data);
  }
  
  private handleWorkerMessage(worker: Worker, data: any): void {
    const task = (worker as any).__currentTask;
    if (task) {
      task.resolve(data.data);
      delete (worker as any).__currentTask;
    }
    
    // Mark worker as available
    this.availableWorkers.push(worker);
    
    // Process next task in queue
    if (this.taskQueue.length > 0) {
      this.runTask(this.taskQueue.shift()!);
    }
  }
  
  private handleWorkerError(worker: Worker, error: ErrorEvent): void {
    const task = (worker as any).__currentTask;
    if (task) {
      task.reject(error);
      delete (worker as any).__currentTask;
    }
    
    this.availableWorkers.push(worker);
  }
  
  terminate(): void {
    this.workers.forEach(worker => worker.terminate());
    this.workers = [];
    this.availableWorkers = [];
    this.taskQueue = [];
  }
}

// Usage
const pool = new WorkerPool(
  new URL('./worker.ts', import.meta.url).toString(),
  4 // 4 workers in pool
);

// Execute tasks in parallel
async function processInParallel() {
  const datasets = [data1, data2, data3, data4, data5];
  
  const results = await Promise.all(
    datasets.map(data => pool.execute('PROCESS', data))
  );
  
  console.log('All results:', results);
}

// Cleanup when done
// pool.terminate();
```

### React Worker Pool Hook

```typescript
// useWorkerPool.ts
import { useEffect, useRef } from 'react';
import { WorkerPool } from './WorkerPool';

export function useWorkerPool(
  workerUrl: string,
  poolSize?: number
) {
  const poolRef = useRef<WorkerPool>();
  
  useEffect(() => {
    poolRef.current = new WorkerPool(workerUrl, poolSize);
    
    return () => {
      poolRef.current?.terminate();
    };
  }, [workerUrl, poolSize]);
  
  const execute = async <T, R>(type: string, data: T): Promise<R> => {
    if (!poolRef.current) {
      throw new Error('Worker pool not initialized');
    }
    return poolRef.current.execute<T, R>(type, data);
  };
  
  return { execute };
}

// Usage
function ParallelProcessor() {
  const { execute } = useWorkerPool(
    new URL('./worker.ts', import.meta.url).toString(),
    4
  );
  
  const handleProcess = async () => {
    const datasets = [
      Array.from({ length: 10000 }, (_, i) => i),
      Array.from({ length: 10000 }, (_, i) => i + 10000),
      Array.from({ length: 10000 }, (_, i) => i + 20000),
      Array.from({ length: 10000 }, (_, i) => i + 30000)
    ];
    
    const startTime = performance.now();
    
    const results = await Promise.all(
      datasets.map(data => execute('PROCESS', data))
    );
    
    const endTime = performance.now();
    console.log(`Processed in ${endTime - startTime}ms`);
    console.log('Results:', results);
  };
  
  return (
    <button onClick={handleProcess}>
      Process in Parallel
    </button>
  );
}
```

## Optimizing JavaScript Execution

### 1. Avoid Layout Thrashing

```typescript
// ❌ Bad: Layout thrashing (read-write-read-write)
function badLayout() {
  elements.forEach(el => {
    const height = el.offsetHeight; // READ (forces layout)
    el.style.height = height + 10 + 'px'; // WRITE
    // Browser must recalculate layout for next read!
  });
}

// ✅ Good: Batch reads, then batch writes
function goodLayout() {
  // Batch all reads
  const heights = elements.map(el => el.offsetHeight);
  
  // Batch all writes
  elements.forEach((el, i) => {
    el.style.height = heights[i] + 10 + 'px';
  });
}

// ✅ Better: Use requestAnimationFrame
function betterLayout() {
  const heights: number[] = [];
  
  // Read phase
  requestAnimationFrame(() => {
    elements.forEach(el => {
      heights.push(el.offsetHeight);
    });
    
    // Write phase (in next frame)
    requestAnimationFrame(() => {
      elements.forEach((el, i) => {
        el.style.height = heights[i] + 10 + 'px';
      });
    });
  });
}
```

### 2. Debounce Expensive Operations

```typescript
// Debounce utility
function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: NodeJS.Timeout;
  
  return function executedFunction(...args: Parameters<T>) {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}

// Usage
const expensiveSearch = debounce((query: string) => {
  // Expensive search operation
  performSearch(query);
}, 300);

// User types: only fires 300ms after they stop typing
searchInput.addEventListener('input', (e) => {
  expensiveSearch((e.target as HTMLInputElement).value);
});
```

### 3. Throttle High-Frequency Events

```typescript
// Throttle utility
function throttle<T extends (...args: any[]) => any>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle: boolean;
  
  return function executedFunction(...args: Parameters<T>) {
    if (!inThrottle) {
      func(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Usage
const handleScroll = throttle(() => {
  // Expensive scroll handler
  updateScrollPosition();
  checkVisibleElements();
}, 100);

window.addEventListener('scroll', handleScroll);
```

### 4. Memoize Expensive Computations

```typescript
// Memoization utility
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();
  
  return ((...args: Parameters<T>) => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// Usage
const expensiveCalculation = memoize((n: number) => {
  console.log('Computing...');
  let result = 0;
  for (let i = 0; i < n; i++) {
    result += Math.sqrt(i);
  }
  return result;
});

expensiveCalculation(1000000); // Computes
expensiveCalculation(1000000); // Returns cached result
```

## Common Mistakes

### 1. Blocking Main Thread with Synchronous Loops

```typescript
// ❌ Bad: Blocks UI for seconds
function processAllItems(items: any[]) {
  for (let i = 0; i < items.length; i++) {
    expensiveOperation(items[i]);
  }
}

// ✅ Good: Chunk processing
async function processAllItemsChunked(items: any[]) {
  for (let i = 0; i < items.length; i += 100) {
    const chunk = items.slice(i, i + 100);
    chunk.forEach(item => expensiveOperation(item));
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}
```

### 2. Not Using Web Workers for Heavy Computation

```typescript
// ❌ Bad: Heavy computation on main thread
function calculatePrimes(max: number) {
  const primes = [];
  for (let i = 2; i < max; i++) {
    if (isPrime(i)) primes.push(i);
  }
  return primes;
}

// ✅ Good: Use Web Worker
// main.ts
const worker = new Worker('./primes.worker.ts');
worker.postMessage({ max: 1000000 });
worker.onmessage = (e) => console.log('Primes:', e.data);

// primes.worker.ts
self.onmessage = (e) => {
  const primes = calculatePrimes(e.data.max);
  self.postMessage(primes);
};
```

### 3. Layout Thrashing

```typescript
// ❌ Bad: Forces multiple reflows
elements.forEach(el => {
  const width = el.offsetWidth; // READ
  el.style.width = width + 10 + 'px'; // WRITE
});

// ✅ Good: Batch reads and writes
const widths = elements.map(el => el.offsetWidth);
elements.forEach((el, i) => {
  el.style.width = widths[i] + 10 + 'px';
});
```

### 4. Not Debouncing/Throttling Event Handlers

```typescript
// ❌ Bad: Fires hundreds of times per second
window.addEventListener('scroll', () => {
  expensiveScrollHandler();
});

// ✅ Good: Throttle to ~10 times per second
window.addEventListener('scroll', throttle(() => {
  expensiveScrollHandler();
}, 100));
```

### 5. Synchronous XHR

```typescript
// ❌ Bad: Blocks main thread
const xhr = new XMLHttpRequest();
xhr.open('GET', '/api/data', false); // false = synchronous!
xhr.send();

// ✅ Good: Async fetch
async function fetchData() {
  const response = await fetch('/api/data');
  return response.json();
}
```

## Best Practices

1. **Break up long tasks** into chunks with setTimeout or requestIdleCallback
2. **Use Web Workers** for CPU-intensive operations (parsing, calculations, image processing)
3. **Monitor long tasks** with PerformanceObserver to identify bottlenecks
4. **Debounce inputs** (search, resize) and throttle high-frequency events (scroll, mousemove)
5. **Batch DOM reads and writes** to avoid layout thrashing
6. **Memoize expensive functions** to avoid redundant calculations
7. **Use requestAnimationFrame** for visual updates and animations
8. **Avoid synchronous operations** (XHR, blocking loops)
9. **Profile with DevTools** to identify performance bottlenecks
10. **Set performance budgets** for JavaScript execution time (<50ms per task)

## When to Use

### Use setTimeout Chunking

- Processing large arrays (>1000 items)
- Iterative calculations that aren't time-sensitive
- Background data processing

### Use requestIdleCallback

- Analytics and tracking
- Lazy loading non-critical resources
- Cache updates
- Prefetching future content

### Use Web Workers

- Image/video processing
- Large data parsing (CSV, JSON)
- Cryptography and hashing
- Complex calculations (physics, simulations)
- Client-side search/filtering

### Use requestAnimationFrame

- Animations
- Visual updates
- Scroll effects
- DOM measurements and updates

## Interview Questions

### 1. What is a long task and why is it problematic?

**Answer**: A long task is any JavaScript execution that blocks the main thread for more than 50ms. It's problematic because during this time, the browser cannot respond to user interactions (clicks, typing, scrolling), making the page feel unresponsive. Long tasks are detected with PerformanceObserver using the 'longtask' entry type. To fix them, break work into smaller chunks using setTimeout, requestIdleCallback, or offload to Web Workers.

### 2. Explain the difference between debouncing and throttling.

**Answer**: Debouncing delays function execution until after a specified wait time has elapsed since the last invocation (useful for search inputs - wait until user stops typing). Throttling ensures a function executes at most once per specified time period, regardless of how many times it's called (useful for scroll handlers - limit to once per 100ms). Debounce = "wait for pause", throttle = "limit frequency".

### 3. What is layout thrashing and how do you prevent it?

**Answer**: Layout thrashing occurs when you repeatedly read and write to the DOM in an interleaved pattern, forcing the browser to recalculate layout multiple times. For example, reading offsetHeight then setting style.height in a loop. Prevent it by batching all DOM reads first, then performing all writes. Use requestAnimationFrame to separate read/write phases, or libraries like fastdom for automatic batching.

### 4. When should you use Web Workers vs requestIdleCallback?

**Answer**: Use Web Workers for CPU-intensive tasks that need to complete quickly without blocking the UI (image processing, data parsing, calculations). Use requestIdleCallback for low-priority tasks that can wait until the browser is idle (analytics, cache updates, prefetching). Workers run in parallel threads, while requestIdleCallback still runs on the main thread but during idle periods.

### 5. How does requestAnimationFrame differ from setTimeout?

**Answer**: requestAnimationFrame is synchronized with the browser's repaint cycle (typically 60fps/16.67ms), ensuring smooth animations and efficient rendering. It automatically pauses when the tab is inactive, saving CPU. setTimeout has no synchronization with rendering and may fire at arbitrary times, potentially causing jank. Always use requestAnimationFrame for visual updates and animations.

### 6. What data can be transferred to Web Workers?

**Answer**: Workers communicate via postMessage using structured clone algorithm. This supports most data types (objects, arrays, primitives, Dates, RegExp, Blob, ArrayBuffer) but not functions, DOM nodes, or objects with methods. For large data, use Transferable Objects (ArrayBuffer, MessagePort) which transfer ownership instead of copying, avoiding serialization overhead.

### 7. How do you identify and fix performance bottlenecks in JavaScript execution?

**Answer**: 
1. Use Chrome DevTools Performance panel to record and analyze flame charts
2. Monitor Long Tasks with PerformanceObserver
3. Check User Timing API marks and measures
4. Look for long yellow bars (scripting) in timeline
5. Fix by: chunking work, using workers, memoizing, debouncing/throttling, optimizing algorithms
6. Measure impact with Lighthouse and Web Vitals

### 8. What is the main thread and why is it important to keep it responsive?

**Answer**: The main thread handles JavaScript execution, style calculation, layout, paint, and user input processing. If blocked by long-running JavaScript, the browser cannot respond to user interactions or update the display, causing poor UX. Keep it responsive by limiting tasks to <50ms, offloading heavy work to Web Workers, and using requestIdleCallback for non-critical tasks. This directly impacts FID (First Input Delay) and INP (Interaction to Next Paint) metrics.

## Key Takeaways

1. **Long tasks (>50ms) block user interactions** - break them into smaller chunks
2. **Main thread handles everything** - JavaScript, rendering, and user input compete for time
3. **Web Workers enable true parallelism** - use for CPU-intensive tasks without blocking UI
4. **requestIdleCallback runs work during idle time** - perfect for non-critical background tasks
5. **Layout thrashing kills performance** - batch DOM reads separately from writes
6. **Debounce inputs, throttle events** - reduce expensive operation frequency
7. **requestAnimationFrame for visual updates** - synchronized with browser rendering
8. **Monitor with PerformanceObserver** - detect and track long tasks in production
9. **Chunk processing with setTimeout(0)** - yield to browser between chunks
10. **Set 50ms task budget** - ensure responsive UI and good Core Web Vitals scores

## Resources

- [Long Tasks API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Long_Tasks_API)
- [requestIdleCallback (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback)
- [Web Workers API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [Optimize Long Tasks (web.dev)](https://web.dev/optimize-long-tasks/)
- [JavaScript Performance Optimization](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution)
- [The Main Thread (Chrome Developers)](https://developer.chrome.com/docs/lighthouse/performance/mainthread-work-breakdown/)
- [Off Main Thread Architecture](https://www.smashingmagazine.com/2016/12/gpu-animation-doing-it-right/)
