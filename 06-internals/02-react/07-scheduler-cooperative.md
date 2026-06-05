# React Scheduler: Cooperative Scheduling Internals

## The Idea

**In plain English:** React's cooperative scheduler is a built-in system that breaks up rendering work into tiny chunks and voluntarily pauses between them, so the browser stays free to respond to your clicks and keep animations smooth. "Cooperative" means the scheduler chooses to give up control on its own — nothing forces it to stop.

**Real-world analogy:** Imagine a chef (the scheduler) working in a busy restaurant kitchen who also has to answer the front desk whenever a customer walks in. Instead of cooking one enormous dish without stopping, the chef sets a 5-minute timer, preps as much as possible, then steps away to check the front desk before coming back to continue cooking.

- The chef = the React Scheduler
- The 5-minute timer = the 5 ms time-slice budget checked by `shouldYieldToHost`
- Prepping dishes = processing fiber (component) units of work in `workLoopConcurrent`
- Checking the front desk = letting the browser handle paint and input events between macrotasks

---

## Table of Contents
- [Overview](#overview)
- [Why a Custom Scheduler](#why-a-custom-scheduler)
- [MessageChannel and the Work Loop](#messagechannel-and-the-work-loop)
- [Task Queue and Priority Buckets](#task-queue-and-priority-buckets)
- [Time Slicing and shouldYield](#time-slicing-and-shouldyield)
- [The performWorkUntilDeadline Loop](#the-performworkuntildeadline-loop)
- [Integration with the React Reconciler](#integration-with-the-react-reconciler)
- [Delay Queue and Timer Tasks](#delay-queue-and-timer-tasks)
- [Profiling and Instrumentation](#profiling-and-instrumentation)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

The React Scheduler (`packages/scheduler`) is a standalone cooperative task scheduler that React uses to interleave reconciliation work with browser rendering. "Cooperative" means the scheduler voluntarily yields control back to the browser at time boundaries, rather than relying on OS pre-emption. This is necessary because JavaScript is single-threaded: long synchronous tasks starve the event loop, causing frame drops, missed input events, and janky animations.

The Scheduler's responsibilities:
1. Maintain a priority-ordered queue of tasks
2. Grant each task a time slice (~5 ms by default)
3. Detect when the time slice expires and yield to the browser
4. Resume tasks from where they left off via a message-passing mechanism
5. Handle delayed tasks (the `delay` option) using `setTimeout`

The Scheduler is intentionally decoupled from React — it is a general-purpose cooperative scheduler that could power any framework.

## Why a Custom Scheduler

### The Problem with Native APIs

```
Candidate API     | Problem
──────────────────────────────────────────────────────────────────────
setTimeout(fn, 0) | 4ms minimum clamp; cannot interleave with input
setInterval       | Same 4ms clamp; drift; cannot yield mid-task
requestAnimationFrame | Runs once per frame (~16ms); tied to display refresh
requestIdleCallback  | Only fires when browser is idle; 50ms budget; not available in all envs; Safari had it removed briefly
Generators/coroutines | Would require wrapping all component code; no performance benefit without JS engine support
Promise microtasks  | Microtasks drain before any macrotask; cannot yield to paint or input
```

React needs sub-frame granularity (yield every ~5 ms) that is reliably available across environments (browser, web workers, Node.js, React Native). `MessageChannel` posts a macrotask with zero artificial delay, which is faster than `setTimeout(fn, 0)` and available in all modern JS environments.

## MessageChannel and the Work Loop

```typescript
// packages/scheduler/src/forks/Scheduler.ts (simplified)

// MessageChannel creates a pair of ports; posting on one delivers a macrotask
// to the other — this is how the scheduler re-enters the event loop between
// frames without the 4ms minimum delay of setTimeout.
let channel: MessageChannel;
let port: MessagePort;

if (typeof MessageChannel !== 'undefined') {
  channel = new MessageChannel();
  port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
} else {
  // Fallback for environments without MessageChannel
  // (rare — Node.js test environments, old browsers)
  (port as any) = {
    postMessage() {
      setTimeout(performWorkUntilDeadline, 0);
    },
  };
}

// Scheduling a task means posting a message — which puts performWorkUntilDeadline
// in the macrotask queue AFTER any pending microtasks and after the browser
// has had a chance to paint / process input.
function schedulePerformWorkUntilDeadline() {
  port.postMessage(null);
}
```

### Why Not a Microtask?

```typescript
// Microtasks (Promise.resolve().then, queueMicrotask) drain BEFORE the browser
// can paint or process input events. If the Scheduler used microtasks, it would
// never actually yield control to the browser — the microtask queue would keep
// running until empty.

// This is what we want:
//  [render task chunk] → yield → [browser: input/paint] → [next render chunk]

// With microtask-based re-scheduling:
//  [render task chunk] → microtask → [render task chunk] → microtask → ...
//  → browser finally gets control only when work queue is empty

// MessageChannel gives us the macrotask re-entry we need.
```

## Task Queue and Priority Buckets

The Scheduler maintains two heaps (min-heaps sorted by `sortIndex`):

1. **`taskQueue`** — tasks that are ready to run now (sorted by `expirationTime`)
2. **`timerQueue`** — tasks that have a `delay > 0` (sorted by `startTime`)

```typescript
// Priority constants
export const ImmediatePriority = 1;   // sortIndex = -1 (always first)
export const UserBlockingPriority = 2; // timeout = 250ms
export const NormalPriority = 3;       // timeout = 5000ms (5 seconds)
export const LowPriority = 4;          // timeout = 10000ms (10 seconds)
export const IdlePriority = 5;         // timeout = maxSafeInt (never expires)

const timeouts: Record<number, number> = {
  [ImmediatePriority]:    -1,      // already expired
  [UserBlockingPriority]: 250,
  [NormalPriority]:       5000,
  [LowPriority]:          10000,
  [IdlePriority]:         1073741823, // ~12 days
};

interface Task {
  id: number;               // monotonically increasing
  callback: Callback | null;
  priorityLevel: PriorityLevel;
  startTime: number;        // when it became eligible to run
  expirationTime: number;   // startTime + timeout[priority]
  sortIndex: number;        // = expirationTime for taskQueue, startTime for timerQueue
}

function scheduleCallback(
  priorityLevel: PriorityLevel,
  callback: Callback,
  options?: { delay?: number },
): Task {
  const currentTime = getCurrentTime();
  const startTime = options?.delay
    ? currentTime + options.delay
    : currentTime;

  const timeout = timeouts[priorityLevel];
  const expirationTime = startTime + timeout;

  const newTask: Task = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };

  if (startTime > currentTime) {
    // Delayed task — goes to timerQueue
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // This is the earliest timer, schedule a setTimeout to promote it
      if (isHostTimeoutScheduled) cancelHostTimeout();
      isHostTimeoutScheduled = true;
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // Ready now — goes to taskQueue
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(); // → schedulePerformWorkUntilDeadline()
    }
  }

  return newTask;
}
```

### Min-Heap Implementation

```typescript
// The Scheduler uses a simple array-backed min-heap.
// This is O(log n) for push/pop — crucial for keeping scheduling overhead minimal.

function push<T extends { sortIndex: number; id: number }>(heap: T[], node: T): void {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}

function pop<T extends { sortIndex: number; id: number }>(heap: T[]): T | null {
  if (heap.length === 0) return null;
  const first = heap[0];
  const last = heap.pop()!;
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);
  }
  return first;
}

// Comparison: lower sortIndex = higher priority
// Tie-break by task.id (older tasks win)
function compare(a: { sortIndex: number; id: number }, b: { sortIndex: number; id: number }): number {
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```

## Time Slicing and shouldYield

The heart of the Scheduler's cooperative behaviour is `shouldYieldToHost`, which checks whether the current time slice has expired.

```typescript
const frameYieldMs = 5; // 5ms time slice
let deadline = 0;

function shouldYieldToHost(): boolean {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameYieldMs) {
    // Still within the time slice — keep going
    return false;
  }
  // Time slice expired
  if (enableIsInputPending) {
    // Check for pending input events using the isInputPending API
    // (available in Chromium via navigator.scheduling.isInputPending)
    if (needsPaint || scheduling.isInputPending(continuousOptions)) {
      return true;
    }
    // If no input pending and time slice is only slightly over budget,
    // extend to 300ms to avoid unnecessary yielding
    if (timeElapsed < continuousYieldMs) {
      return false;
    }
    if (timeElapsed < maxYieldMs) {
      return scheduling.isInputPending(discreteOptions) ? true : false;
    }
  }
  return true; // Always yield after 300ms regardless
}

export { shouldYieldToHost as unstable_shouldYield };
```

### How React Uses shouldYield

```typescript
// packages/react-reconciler/src/ReactFiberWorkLoop.ts

function workLoopConcurrent() {
  // Process fibers in the work-in-progress tree until:
  // 1. The tree is complete (workInProgress === null), OR
  // 2. The scheduler says we should yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

// Each call to performUnitOfWork processes one fiber (one component).
// The loop is broken at fiber granularity — React cannot yield mid-component.
// This is why render functions should be cheap; a slow component blocks the loop.
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  let next: Fiber | null;

  next = beginWork(current, unitOfWork, renderLanes);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // No child — complete this fiber
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

## The performWorkUntilDeadline Loop

This is the function that runs when the MessageChannel delivers a message. It runs as many tasks as possible within the current time slice, then either posts another message (to continue work) or stops.

```typescript
let scheduledHostCallback: (
  hasTimeRemaining: boolean,
  initialTime: number,
) => boolean | null = null;
let isMessageLoopRunning = false;

const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // Establish the deadline for this slice
    deadline = currentTime + frameYieldMs;
    const hasTimeRemaining = true;

    let hasMoreWork = true;
    try {
      // Run React's work loop, passing the current time and whether time remains
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        // More tasks remain — post another message to re-enter in the next macrotask
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
};

// The callback passed to scheduledHostCallback is flushWork:
function flushWork(hasTimeRemaining: boolean, initialTime: number): boolean {
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }

  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
  }
}
```

### The Inner workLoop

```typescript
function workLoop(hasTimeRemaining: boolean, initialTime: number): boolean {
  let currentTime = initialTime;
  // Check if any delayed tasks are now ready
  advanceTimers(currentTime);

  currentTask = peek(taskQueue);

  while (currentTask !== null && !(enableSchedulerDebugging && isSchedulerPaused)) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // This task hasn't expired yet and we're out of time — yield
      break;
    }

    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;

      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      // Call the task. If it returns a function, it has more work to do.
      const continuationCallback = callback(didUserCallbackTimeout);

      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        // Task yielded — put the continuation back in the queue
        currentTask.callback = continuationCallback;
      } else {
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      advanceTimers(currentTime);
    } else {
      // Callback was cancelled (set to null by cancelCallback)
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }

  // Return true if there's more work
  if (currentTask !== null) {
    return true;
  }
  // Check timerQueue for future work
  const firstTimer = peek(timerQueue);
  if (firstTimer !== null) {
    requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
  }
  return false;
}
```

## Integration with the React Reconciler

The bridge between the Scheduler package and the React reconciler is in `ReactFiberRootScheduler.ts`.

```typescript
// Simplified ReactFiberRootScheduler.ts

import {
  scheduleCallback,
  cancelCallback,
  NormalPriority as NormalSchedulerPriority,
  UserBlockingPriority as UserBlockingSchedulerPriority,
  ImmediatePriority as ImmediateSchedulerPriority,
} from 'scheduler';

// When React has work to do, it calls ensureRootIsScheduled
export function ensureRootIsScheduled(root: FiberRoot): void {
  const existingCallbackNode = root.callbackNode;
  const nextLanes = getNextLanes(root, NoLanes);

  if (nextLanes === NoLanes) {
    // No work — cancel any existing scheduled task
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
      root.callbackNode = null;
      root.callbackPriority = NoLane;
    }
    return;
  }

  // Determine scheduler priority from lane priority
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  // If we already have a task scheduled at the same priority, reuse it
  const existingCallbackPriority = root.callbackPriority;
  if (existingCallbackPriority === newCallbackPriority) {
    return; // nothing to do
  }

  if (existingCallbackNode !== null) {
    cancelCallback(existingCallbackNode);
  }

  // Schedule a new task
  let newCallbackNode: unknown;
  if (newCallbackPriority === SyncLane) {
    // Sync work is not scheduled through the scheduler — it runs immediately
    // (or is batched into a microtask via scheduleMicrotask)
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    scheduleMicrotask(flushSyncCallbacks);
    newCallbackNode = null;
  } else {
    const schedulerPriorityLevel = lanesToSchedulerPriority(newCallbackPriority);
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

### performConcurrentWorkOnRoot as a Continuation

```typescript
function performConcurrentWorkOnRoot(
  root: FiberRoot,
  didTimeout: boolean,
): RenderTaskFn | null {
  // ... setup ...

  // Determine if we should use time-sliced rendering or synchronous rendering
  const shouldTimeSlice =
    !includesBlockingLane(root, lanes) &&
    !includesExpiredLane(root, lanes) &&
    (disableSchedulerTimeoutInWorkLoop || !didTimeout);

  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)   // uses workLoopConcurrent (yields)
    : renderRootSync(root, lanes);        // uses workLoopSync (no yields)

  // If rendering was interrupted (yielded), return a continuation callback.
  // The Scheduler will store this continuation and call it on the next task.
  if (exitStatus === RootInProgress) {
    // Work was interrupted — return this function as a continuation
    return performConcurrentWorkOnRoot.bind(null, root);
  }

  // Rendering completed — commit
  // ...
  return null;
}
```

This is the crucial point: `performConcurrentWorkOnRoot` returns itself as a continuation when work is incomplete. The Scheduler's `workLoop` sees a function return value and stores it back in `currentTask.callback`, so the same logical task resumes on the next `performWorkUntilDeadline` invocation.

## Delay Queue and Timer Tasks

```typescript
// Scheduling deferred work with a delay
const task = scheduleCallback(
  LowPriority,
  () => { prefetchData(); },
  { delay: 500 }, // run 500ms from now
);

// Cancel if no longer needed
cancelCallback(task);

// Internal: advanceTimers moves tasks from timerQueue to taskQueue
function advanceTimers(currentTime: number): void {
  let timer = peek(timerQueue);
  while (timer !== null) {
    if (timer.callback === null) {
      // Cancelled timer
      pop(timerQueue);
    } else if (timer.startTime <= currentTime) {
      // Timer has fired — move to taskQueue
      pop(timerQueue);
      timer.sortIndex = timer.expirationTime;
      push(taskQueue, timer);
    } else {
      // Next timer is in the future — stop
      break;
    }
    timer = peek(timerQueue);
  }
}

// handleTimeout is called by setTimeout when the earliest timer fires
function handleTimeout(currentTime: number): void {
  isHostTimeoutScheduled = false;
  advanceTimers(currentTime);

  if (!isHostCallbackScheduled) {
    if (peek(taskQueue) !== null) {
      isHostCallbackScheduled = true;
      requestHostCallback();
    } else {
      // More timers pending
      const firstTimer = peek(timerQueue);
      if (firstTimer !== null) {
        requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
      }
    }
  }
}
```

## Profiling and Instrumentation

The Scheduler has a built-in profiling mode that logs task lifecycle events. React DevTools uses this data to populate the Profiler flamegraph.

```typescript
// Enable profiling (enabled by default in development builds)
// In production, all profiling calls are no-ops.

interface ProfilingEvent {
  type: 'task-start' | 'task-complete' | 'task-cancel' | 'task-error' | 'schedule';
  taskId: number;
  priorityLevel: PriorityLevel;
  time: number;
}

// Profiler hooks used by React DevTools
export const unstable_Profiling = enableProfiling
  ? {
      startLoggingProfilingEvents(): void { /* ... */ },
      stopLoggingProfilingEvents(): ArrayBuffer { /* ... */ },
    }
  : null;

// React DevTools reads these to show task duration and priority in the Profiler tab
```

## Common Pitfalls

### Pitfall 1: Long Synchronous Operations Inside Render

```typescript
// BAD: synchronous expensive computation blocks the work loop
function ExpensiveList({ items }: { items: Item[] }) {
  // This runs synchronously inside performUnitOfWork
  // The work loop cannot yield between items in the same render function
  const sorted = items.sort((a, b) => expensiveCompare(a, b)); // O(n log n), slow

  return <ul>{sorted.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// BETTER: Memoize the expensive computation
function BetterList({ items }: { items: Item[] }) {
  const sorted = useMemo(
    () => [...items].sort((a, b) => expensiveCompare(a, b)),
    [items],
  );
  return <ul>{sorted.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// BEST for very large lists: virtualize + use a transition
function BestList({ items }: { items: Item[] }) {
  const [sortedItems, setSortedItems] = useState(items);
  useEffect(() => {
    startTransition(() => {
      setSortedItems([...items].sort((a, b) => expensiveCompare(a, b)));
    });
  }, [items]);
  return <VirtualList items={sortedItems} />;
}
```

### Pitfall 2: Misunderstanding the 5ms Frame Budget

```typescript
// The 5ms budget is per Scheduler task slice, not per component.
// React processes as many fibers as possible within 5ms,
// then yields. There is no per-component budget.

// This means a tree of 10,000 tiny components can be processed in
// a few slices (fast). But ONE component with a 10ms render blocks the
// entire 5ms slice and forces React to overshoot the deadline.

// Component render time matters more than component count.
```

### Pitfall 3: Calling scheduleCallback Directly

```typescript
// Don't use the scheduler package directly in application code.
// It is an internal implementation detail. The API is unstable_*.

// BAD:
import { unstable_scheduleCallback, unstable_NormalPriority } from 'scheduler';
unstable_scheduleCallback(unstable_NormalPriority, myWork);

// GOOD: Use React's public APIs
startTransition(myWork);       // for UI updates
useTransition()[1](myWork);    // same, with isPending
useDeferredValue(value);       // for deferred values
setTimeout/requestIdleCallback // for non-React work
```

### Pitfall 4: Expecting Consistent Task Ordering

```typescript
// The Scheduler is a priority queue, not a FIFO queue.
// Two tasks at the same priority are ordered by expiration time,
// then by task id (insertion order). But a higher-priority task
// inserted later will always run before a lower-priority task.

scheduleCallback(LowPriority, taskA);
scheduleCallback(NormalPriority, taskB); // Will run before taskA despite being scheduled later
scheduleCallback(LowPriority, taskC);    // Will run after taskB but before/after taskA by expiration
```

### Pitfall 5: Forgetting That shouldYield Checks at Fiber Granularity

```typescript
// React can only yield BETWEEN fibers (between component render calls).
// A single render function that takes 50ms will block for 50ms regardless
// of what the scheduler says. There is no way for the scheduler to
// interrupt a running JS function mid-execution.

// This is why "render functions should be fast" is a core React principle.
// It's not just a style preference — it's a hard requirement for cooperative scheduling.
```

## Interview Questions

### Question 1: Why does React's Scheduler use MessageChannel instead of setTimeout?

**Answer**: `setTimeout(fn, 0)` has a 4ms minimum delay clamped by browsers (and up to 1 second when the tab is in the background). `MessageChannel.port.postMessage` posts a macrotask with no artificial minimum delay, executing as fast as the browser's task queue allows — typically sub-millisecond in idle conditions. More importantly, both are macrotasks (unlike `Promise.then` which is a microtask), which means the browser can process pending paint and input events between consecutive scheduler messages. This is what makes the scheduling cooperative.

### Question 2: How does React prevent a transition render from running forever?

**Answer**: When React returns `performConcurrentWorkOnRoot` as a continuation, the Scheduler stores it as `currentTask.callback`. On each subsequent `performWorkUntilDeadline` call, the Scheduler's `workLoop` checks if the task has exceeded its expiration time (`task.expirationTime <= currentTime`). If so, it passes `didTimeout = true` to the callback. React's `performConcurrentWorkOnRoot` sees `didTimeout = true` and switches to synchronous (non-yielding) rendering for that commit. Separately, `markStarvedLanesAsExpired` in the reconciler promotes the lane to `expiredLanes`, ensuring the render runs synchronously. This guarantees transitions always eventually complete.

### Question 3: What is the difference between the taskQueue and the timerQueue?

**Answer**: The `taskQueue` is a min-heap of tasks that are ready to run now, sorted by expiration time. The `timerQueue` is a min-heap of tasks with a future `delay`, sorted by their `startTime`. Each time the work loop runs, it calls `advanceTimers` to check if any timer has reached its `startTime` and moves those tasks from the `timerQueue` to the `taskQueue`. A `setTimeout` is registered to wake up the Scheduler when the earliest timer in the `timerQueue` fires.

### Question 4: Explain how a continuation callback enables incremental rendering.

**Answer**: `scheduleCallback` stores the return value of a task callback back as the task's new callback if it is a function. `performConcurrentWorkOnRoot` returns itself (partially applied with the root) when the reconciler's work loop is interrupted by `shouldYield()`. This means the same Scheduler task "continues" on the next `performWorkUntilDeadline` invocation, resuming reconciliation from the fiber that was in progress (`workInProgress` is preserved in the fiber tree). No work is lost — the task just pauses between macrotasks so the browser can paint and process input.

### Question 5: How does the Scheduler handle priority inversion?

**Answer**: The Scheduler does not directly handle priority inversion — it is a straightforward priority queue (min-heap). Priority inversion prevention in React comes from the Lanes model: lanes have expiration times, and when a low-priority lane expires it is promoted to `expiredLanes`, causing React to render it synchronously. The Scheduler priority of the corresponding task is already high enough (it was bumped up when expiration was detected). There is no traditional priority inheritance mechanism; instead, the combination of expiration and synchronous flushing ensures low-priority work cannot be starved indefinitely.

## Key Takeaways

1. **MessageChannel over setTimeout**: Zero-delay macrotask re-entry without the 4ms clamp; allows browser to interleave paint and input handling between chunks

2. **5ms time slice**: `shouldYieldToHost` checks elapsed time and returns true after ~5ms, causing `workLoopConcurrent` to break its fiber-processing loop

3. **Two heaps**: `taskQueue` (ready tasks, sorted by expiration) and `timerQueue` (delayed tasks, sorted by start time) — tasks graduate from timer to task queue via `advanceTimers`

4. **Continuation callbacks**: `performConcurrentWorkOnRoot` returns itself to signal "more work remains" — the Scheduler stores this as the task's new callback, resuming on the next slice

5. **Cooperative, not pre-emptive**: JS is single-threaded; the Scheduler can only yield between fiber units of work, never mid-function — slow components block regardless

6. **Priority is expiration-based**: Lower-priority tasks have longer timeouts before expiring, not a lower CPU share — they simply wait in the queue longer

7. **Scheduler is decoupled from React**: It knows nothing about fibers, components, or state; it only manages `{ callback, priorityLevel, expirationTime }` task objects

8. **SyncLane bypasses the Scheduler**: Synchronous updates use `scheduleMicrotask` (queueMicrotask) for immediate flushing, skipping the Scheduler entirely

## Resources

- [React Scheduler source](https://github.com/facebook/react/tree/main/packages/scheduler/src)
- [React Fiber Architecture (Acdlite)](https://github.com/acdlite/react-fiber-architecture)
- ["Inside Fiber: in-depth overview of the new reconciliation algorithm in React" — Max Koretskyi](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
- [isInputPending API (WICG)](https://github.com/WICG/is-input-pending)
- ["Scheduling internals" React team discussion](https://github.com/facebook/react/issues/13206)
- [Lin Clark's "A Cartoon Intro to Fiber" (React Conf 2017)](https://www.youtube.com/watch?v=ZCuYPiUIONs)
