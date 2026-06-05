# Event Loop, Microtasks, and Macrotasks

## The Idea

**In plain English:** JavaScript can only do one thing at a time, but the event loop is the system that decides what to do next — sorting jobs into a priority list so that urgent tasks (like promise results) always run before less urgent ones (like timers).

**Real-world analogy:** Imagine a restaurant with a single chef (JavaScript's one thread). The chef works through a ticket rail one order at a time. When a ticket arrives, any sauce requests written on sticky notes attached to that ticket (microtasks) must be handled before the chef picks up the next ticket from the rail (macrotasks). A dinner reservation reminder that comes in by phone (a timer like setTimeout) goes to the bottom of a separate pile and can only be handled once the current ticket and all its sticky notes are done.

- The chef = JavaScript's single thread running one task at a time
- A ticket on the rail = a macrotask (e.g., a setTimeout callback or a click event handler)
- A sticky note attached to the ticket = a microtask (e.g., a Promise .then() callback)
- The phone reminder pile = the macrotask queue waiting its turn

---

## Learning Objectives
- Understand how the JavaScript event loop coordinates asynchronous execution
- Master the distinction between microtasks and macrotasks (task queues)
- Learn task queue priority and execution order
- Comprehend how promises, setTimeout, and other async APIs interact
- Understand event loop phases and their implications for performance

## Introduction

JavaScript is **single-threaded**, meaning it executes one piece of code at a time on the call stack. Yet it handles asynchronous operations like network requests, timers, and user interactions efficiently. This is possible through the **event loop**, a mechanism that coordinates the execution of code, collection of events, and processing of queued tasks.

The event loop doesn't live in the JavaScript engine itself (V8, SpiderMonkey, etc.) but is part of the **runtime environment** (browser, Node.js). It manages how and when asynchronous callbacks execute, using different task queues with distinct priorities.

## The Event Loop Architecture

### Core Components

```
┌───────────────────────────┐
│      Call Stack           │  ← JavaScript execution
│  (Synchronous Code)       │
└───────────────────────────┘
            ↑
            │
┌───────────────────────────┐
│      Event Loop           │  ← Orchestrator
│  (Checks queues, pushes   │
│   callbacks to stack)     │
└───────────────────────────┘
     ↑              ↑
     │              │
┌─────────┐  ┌──────────────┐
│Microtask│  │  Macrotask   │
│  Queue  │  │    Queue     │
└─────────┘  └──────────────┘
     ↑              ↑
     │              │
┌─────────┐  ┌──────────────┐
│Promises │  │  setTimeout  │
│async/   │  │  setInterval │
│await    │  │  I/O         │
│queueMic │  │  UI Render   │
│rotask   │  │  requestAF   │
└─────────┘  └──────────────┘
```

### The Event Loop Process

```javascript
while (eventLoop.waitForTask()) {
  // 1. Execute all synchronous code (call stack)
  const task = eventLoop.getNextTask();
  execute(task);
  
  // 2. Execute ALL microtasks
  while (microtaskQueue.hasTasks()) {
    const microtask = microtaskQueue.getNextTask();
    execute(microtask);
  }
  
  // 3. Render updates (in browsers)
  if (needsRendering()) {
    render();
  }
  
  // 4. Get next macrotask (if any)
  // Loop continues...
}
```

## Macrotasks (Task Queue)

**Macrotasks** (also called "tasks") are queued operations that execute one per event loop iteration.

### Sources of Macrotasks

1. **setTimeout / setInterval**
2. **setImmediate** (Node.js only)
3. **I/O operations**
4. **UI rendering events**
5. **Script loading**
6. **User interaction events** (click, scroll, etc.)
7. **requestAnimationFrame** (browser, runs before render)

### setTimeout Example

```javascript
console.log('1: Synchronous');

setTimeout(() => {
  console.log('2: Macrotask (setTimeout)');
}, 0);

console.log('3: Synchronous');

// Output:
// 1: Synchronous
// 3: Synchronous
// 2: Macrotask (setTimeout)
```

**Why?** Even with `0ms` delay, setTimeout schedules a macrotask that executes AFTER all synchronous code completes.

**Execution flow:**

```
1. Execute sync: console.log('1: Synchronous')
2. Schedule setTimeout callback in macrotask queue
3. Execute sync: console.log('3: Synchronous')
4. Call stack empty, event loop checks macrotask queue
5. Execute setTimeout callback: console.log('2: Macrotask')
```

## Microtasks (Job Queue)

**Microtasks** have higher priority than macrotasks. ALL microtasks execute before the next macrotask.

### Sources of Microtasks

1. **Promise callbacks** (.then, .catch, .finally)
2. **async/await** (under the hood uses promises)
3. **queueMicrotask()**
4. **MutationObserver** (browser)
5. **process.nextTick()** (Node.js - technically higher priority than standard microtasks)

### Promise Example

```javascript
console.log('1: Synchronous');

setTimeout(() => {
  console.log('2: Macrotask (setTimeout)');
}, 0);

Promise.resolve()
  .then(() => {
    console.log('3: Microtask (Promise)');
  });

console.log('4: Synchronous');

// Output:
// 1: Synchronous
// 4: Synchronous
// 3: Microtask (Promise)
// 2: Macrotask (setTimeout)
```

**Execution flow:**

```
1. Execute sync: console.log('1: Synchronous')
2. Schedule setTimeout in macrotask queue
3. Promise.resolve() - schedule .then() in microtask queue
4. Execute sync: console.log('4: Synchronous')
5. Call stack empty, process ALL microtasks
   → Execute: console.log('3: Microtask')
6. Microtask queue empty, get next macrotask
   → Execute: console.log('2: Macrotask')
```

## Priority and Execution Order

### The Golden Rule

```
Synchronous Code → ALL Microtasks → ONE Macrotask → Repeat
```

### Complex Example

```javascript
console.log('Start');

setTimeout(() => {
  console.log('setTimeout 1');
  Promise.resolve().then(() => {
    console.log('Promise inside setTimeout 1');
  });
}, 0);

Promise.resolve()
  .then(() => {
    console.log('Promise 1');
    setTimeout(() => {
      console.log('setTimeout inside Promise 1');
    }, 0);
  })
  .then(() => {
    console.log('Promise 2');
  });

setTimeout(() => {
  console.log('setTimeout 2');
}, 0);

console.log('End');

// Output:
// Start
// End
// Promise 1
// Promise 2
// setTimeout 1
// Promise inside setTimeout 1
// setTimeout 2
// setTimeout inside Promise 1
```

**Step-by-step breakdown:**

```
PHASE 1: Synchronous Execution
─────────────────────────────
Call Stack: [console.log('Start')]
Output: "Start"

Call Stack: [setTimeout callback 1 → Macrotask Queue]
Call Stack: [Promise.resolve().then() → Microtask Queue]
Call Stack: [setTimeout callback 2 → Macrotask Queue]
Call Stack: [console.log('End')]
Output: "End"

State:
  Macrotask Queue: [setTimeout 1, setTimeout 2]
  Microtask Queue: [Promise 1 chain]

PHASE 2: Microtask Processing
──────────────────────────────
Execute: Promise 1 callback
Output: "Promise 1"
  → Schedules setTimeout inside Promise 1 → Macrotask Queue
  → Schedules Promise 2 → Microtask Queue

Execute: Promise 2 callback
Output: "Promise 2"

State:
  Macrotask Queue: [setTimeout 1, setTimeout 2, setTimeout inside Promise 1]
  Microtask Queue: []

PHASE 3: First Macrotask
────────────────────────
Execute: setTimeout 1 callback
Output: "setTimeout 1"
  → Schedules Promise inside setTimeout 1 → Microtask Queue

Process microtasks before next macrotask:
Execute: Promise inside setTimeout 1
Output: "Promise inside setTimeout 1"

State:
  Macrotask Queue: [setTimeout 2, setTimeout inside Promise 1]
  Microtask Queue: []

PHASE 4: Second Macrotask
─────────────────────────
Execute: setTimeout 2 callback
Output: "setTimeout 2"

No microtasks scheduled

State:
  Macrotask Queue: [setTimeout inside Promise 1]
  Microtask Queue: []

PHASE 5: Third Macrotask
────────────────────────
Execute: setTimeout inside Promise 1
Output: "setTimeout inside Promise 1"
```

## async/await and the Event Loop

`async/await` is syntactic sugar over promises, so it follows microtask rules.

```javascript
async function asyncFunc() {
  console.log('1: async function start');
  
  await Promise.resolve();
  
  console.log('2: after await');
}

console.log('3: before asyncFunc');
asyncFunc();
console.log('4: after asyncFunc call');

// Output:
// 3: before asyncFunc
// 1: async function start
// 4: after asyncFunc call
// 2: after await
```

**Why?** 

```javascript
// This async/await code:
async function asyncFunc() {
  console.log('1');
  await Promise.resolve();
  console.log('2');
}

// Is equivalent to:
function asyncFunc() {
  return Promise.resolve()
    .then(() => {
      console.log('1');
      return Promise.resolve();
    })
    .then(() => {
      console.log('2');
    });
}
```

Code after `await` runs in a microtask, not synchronously.

### Multiple awaits

```javascript
async function example() {
  console.log('1: Start');
  
  await Promise.resolve();
  console.log('2: After first await');
  
  await Promise.resolve();
  console.log('3: After second await');
}

console.log('4: Before example()');
example();
console.log('5: After example() call');

setTimeout(() => console.log('6: setTimeout'), 0);

// Output:
// 4: Before example()
// 1: Start
// 5: After example() call
// 2: After first await
// 3: After second await
// 6: setTimeout
```

Each `await` creates a new microtask for subsequent code.

## queueMicrotask API

Explicit microtask scheduling without promises:

```javascript
console.log('1: Synchronous');

queueMicrotask(() => {
  console.log('2: Microtask via queueMicrotask');
});

Promise.resolve().then(() => {
  console.log('3: Microtask via Promise');
});

setTimeout(() => {
  console.log('4: Macrotask via setTimeout');
}, 0);

console.log('5: Synchronous');

// Output:
// 1: Synchronous
// 5: Synchronous
// 2: Microtask via queueMicrotask
// 3: Microtask via Promise
// 4: Macrotask via setTimeout
```

## Node.js Event Loop Phases

Node.js has a more complex event loop with multiple phases:

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout/setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred to next loop
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  Retrieve new I/O events; execute I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

### process.nextTick() vs setImmediate()

```javascript
// Node.js specific
setImmediate(() => {
  console.log('1: setImmediate (check phase)');
});

process.nextTick(() => {
  console.log('2: process.nextTick (before any phase)');
});

Promise.resolve().then(() => {
  console.log('3: Promise (microtask)');
});

// Output:
// 2: process.nextTick
// 3: Promise
// 1: setImmediate
```

**process.nextTick()** has higher priority than microtasks and executes before ANY event loop phase.

**Order in Node.js:**
1. process.nextTick() queue
2. Microtask queue (promises)
3. Event loop phases (timers, poll, check, etc.)

```javascript
setTimeout(() => console.log('1: setTimeout'), 0);

setImmediate(() => console.log('2: setImmediate'));

process.nextTick(() => console.log('3: nextTick'));

Promise.resolve().then(() => console.log('4: Promise'));

console.log('5: Synchronous');

// Output:
// 5: Synchronous
// 3: nextTick
// 4: Promise
// 1: setTimeout
// 2: setImmediate
```

## Microtask Queue Starvation

If microtasks continuously add more microtasks, macrotasks never execute:

```javascript
console.log('Start');

setTimeout(() => {
  console.log('This will NEVER execute!');
}, 0);

function addMicrotask() {
  Promise.resolve().then(() => {
    console.log('Microtask');
    addMicrotask(); // Infinite microtask loop!
  });
}

addMicrotask();

console.log('End');

// Output:
// Start
// End
// Microtask
// Microtask
// Microtask
// ... (infinite, setTimeout never runs)
```

**Why?** The event loop processes ALL microtasks before moving to the next macrotask. If microtasks keep spawning, the macrotask queue is starved.

**Solution:** Use setTimeout to break the microtask chain:

```javascript
function addTask() {
  setTimeout(() => {
    console.log('Macrotask');
    addTask(); // Now allows other tasks to execute
  }, 0);
}
```

## Browser Rendering and the Event Loop

Browsers render updates AFTER microtasks, BEFORE the next macrotask:

```
Macrotask → ALL Microtasks → Rendering → Next Macrotask
```

### Example: Blocking Render with Microtasks

```javascript
const button = document.getElementById('myButton');

button.addEventListener('click', () => {
  // This change won't be visible until ALL microtasks complete
  button.textContent = 'Loading...';
  
  // Schedule 1000 microtasks
  for (let i = 0; i < 1000; i++) {
    Promise.resolve().then(() => {
      // Expensive computation
      Math.random();
    });
  }
  
  // This microtask executes after all 1000 above
  Promise.resolve().then(() => {
    button.textContent = 'Done!';
  });
  
  // User won't see "Loading..." because rendering happens
  // AFTER all microtasks complete
});
```

**Solution:** Use setTimeout to allow rendering:

```javascript
button.addEventListener('click', () => {
  button.textContent = 'Loading...';
  
  // Allow render to happen
  setTimeout(() => {
    // Expensive computation
    for (let i = 0; i < 1000000; i++) {
      Math.random();
    }
    button.textContent = 'Done!';
  }, 0);
  
  // Now "Loading..." is visible before computation
});
```

## requestAnimationFrame

`requestAnimationFrame` (rAF) executes before rendering, but after microtasks:

```
Macrotask → Microtasks → requestAnimationFrame → Render → Next Macrotask
```

```javascript
console.log('1: Synchronous');

setTimeout(() => {
  console.log('2: setTimeout (macrotask)');
}, 0);

requestAnimationFrame(() => {
  console.log('3: rAF (before render)');
});

Promise.resolve().then(() => {
  console.log('4: Promise (microtask)');
});

// Output:
// 1: Synchronous
// 4: Promise (microtask)
// 3: rAF (before render)
// 2: setTimeout (macrotask)
```

Use rAF for smooth animations:

```javascript
function animate() {
  // Update animation state
  box.style.left = `${position}px`;
  position++;
  
  // Schedule next frame
  requestAnimationFrame(animate);
}

requestAnimationFrame(animate); // Syncs with display refresh rate (60fps)
```

## Common Patterns and Pitfalls

### Pattern 1: Debouncing with setTimeout

```javascript
let timeoutId;

function debounce(callback, delay) {
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      callback.apply(this, args);
    }, delay);
  };
}

const debouncedSearch = debounce((query) => {
  console.log('Searching for:', query);
}, 300);

// Only last call executes after 300ms of inactivity
debouncedSearch('a');
debouncedSearch('ab');
debouncedSearch('abc'); // Only this executes
```

### Pattern 2: Throttling with setTimeout

```javascript
let isThrottled = false;

function throttle(callback, delay) {
  return function(...args) {
    if (isThrottled) return;
    
    callback.apply(this, args);
    isThrottled = true;
    
    setTimeout(() => {
      isThrottled = false;
    }, delay);
  };
}

const throttledScroll = throttle(() => {
  console.log('Scroll handler');
}, 100);

// Executes at most once per 100ms
window.addEventListener('scroll', throttledScroll);
```

### Pattern 3: Microtask for State Batching

```javascript
let stateUpdates = [];
let isScheduled = false;

function setState(update) {
  stateUpdates.push(update);
  
  if (!isScheduled) {
    isScheduled = true;
    queueMicrotask(() => {
      // Batch all state updates
      const updates = stateUpdates;
      stateUpdates = [];
      isScheduled = false;
      
      applyUpdates(updates);
    });
  }
}

// Multiple setState calls batch into single update
setState({ count: 1 });
setState({ count: 2 });
setState({ count: 3 });
// All three batch together in next microtask
```

### Pitfall 1: setTimeout(0) is NOT 0ms

```javascript
console.time('setTimeout');

setTimeout(() => {
  console.timeEnd('setTimeout');
}, 0);

// Output: setTimeout: 4-5ms (actual delay)
```

**Why?**
- Browsers enforce minimum timeout (4ms in Chrome)
- setTimeout is a macrotask, waits for current execution to complete
- Nested setTimeout increases delay

### Pitfall 2: Mixing sync and async code

```javascript
let value = 1;

Promise.resolve().then(() => {
  value = 2;
});

console.log(value); // 1, not 2!

// Promise callback runs in microtask AFTER this synchronous code
```

**Solution:** Use async/await for sequential logic:

```javascript
let value = 1;

async function example() {
  await Promise.resolve();
  value = 2;
  console.log(value); // 2
}

example();
```

### Pitfall 3: Lost error handling

```javascript
setTimeout(() => {
  throw new Error('This error is lost!');
}, 0);

// Error occurs in a different execution context
// try/catch around setTimeout won't catch it

try {
  setTimeout(() => {
    throw new Error('Not caught!');
  }, 0);
} catch (error) {
  console.log('Never executed');
}
```

**Solution:** Handle errors inside async callback:

```javascript
setTimeout(() => {
  try {
    throw new Error('Caught!');
  } catch (error) {
    console.error('Error handled:', error.message);
  }
}, 0);
```

## Performance Implications

### 1. Microtask Queue Blocking

```javascript
// Bad: Blocks event loop
function processLargeArray(array) {
  return array.map(item => {
    return Promise.resolve(item * 2);
  });
}

// All promises create microtasks that execute before next macrotask
await Promise.all(processLargeArray(largeArray)); // Blocks rendering

// Good: Break into chunks
async function processInChunks(array, chunkSize = 1000) {
  const results = [];
  
  for (let i = 0; i < array.length; i += chunkSize) {
    const chunk = array.slice(i, i + chunkSize);
    const chunkResults = await Promise.all(
      chunk.map(item => Promise.resolve(item * 2))
    );
    results.push(...chunkResults);
    
    // Yield to event loop every chunk
    await new Promise(resolve => setTimeout(resolve, 0));
  }
  
  return results;
}
```

### 2. setTimeout Overhead

```javascript
// Bad: setTimeout has ~4ms minimum delay
for (let i = 0; i < 1000; i++) {
  setTimeout(() => processItem(i), 0); // 1000 * 4ms = 4 seconds!
}

// Better: queueMicrotask (no delay)
for (let i = 0; i < 1000; i++) {
  queueMicrotask(() => processItem(i)); // Much faster, but can block
}

// Best: Batch processing
async function processBatch(items) {
  for (let i = 0; i < items.length; i++) {
    processItem(items[i]);
    
    // Yield every 100 items
    if (i % 100 === 0) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}
```

### 3. requestAnimationFrame for Animations

```javascript
// Bad: setTimeout doesn't sync with display refresh
function animateWithTimeout() {
  updateAnimation();
  setTimeout(animateWithTimeout, 16); // ~60fps, but not synced
}

// Good: rAF syncs with display refresh
function animateWithRAF() {
  updateAnimation();
  requestAnimationFrame(animateWithRAF); // Smooth 60fps
}
```

### 4. Measuring Async Performance

```javascript
// Measure event loop delay
function measureEventLoopLag() {
  const start = Date.now();
  
  setTimeout(() => {
    const lag = Date.now() - start;
    console.log(`Event loop lag: ${lag}ms`);
  }, 0);
}

// Check periodically
setInterval(measureEventLoopLag, 1000);

// High lag indicates blocked event loop
```

## Common Misconceptions

### ❌ Misconception 1: setTimeout(fn, 0) executes immediately

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');

// Output: 1, 3, 2 (not 1, 2, 3)
```

**✅ Reality:** setTimeout schedules a macrotask that executes AFTER all synchronous code and microtasks.

### ❌ Misconception 2: Promises are synchronous

```javascript
console.log('1');
Promise.resolve().then(() => console.log('2'));
console.log('3');

// Output: 1, 3, 2 (not 1, 2, 3)
```

**✅ Reality:** Promise .then() callbacks are microtasks that execute after synchronous code.

### ❌ Misconception 3: All async code has the same priority

```javascript
setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));

// Output: Promise, setTimeout (not setTimeout, Promise)
```

**✅ Reality:** Microtasks (promises) have higher priority than macrotasks (setTimeout).

### ❌ Misconception 4: async makes code run in parallel

```javascript
async function slow() {
  await heavyComputation(); // Still blocks event loop!
}
```

**✅ Reality:** async/await doesn't create threads. CPU-intensive code still blocks. Use Web Workers for true parallelism.

### ❌ Misconception 5: Event loop only exists in browsers

**✅ Reality:** Node.js also has an event loop (with different phases and APIs like setImmediate, process.nextTick).

## Interview Questions

**Q1: Explain the output order of this code:**

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve()
  .then(() => console.log('3'))
  .then(() => console.log('4'));

console.log('5');
```

**A:** Output: 1, 5, 3, 4, 2

**Explanation:**
1. Synchronous: `console.log('1')` → Output: **1**
2. Schedule setTimeout callback in macrotask queue
3. Schedule Promise .then() in microtask queue
4. Synchronous: `console.log('5')` → Output: **5**
5. Call stack empty, process ALL microtasks:
   - Execute first .then() → Output: **3**
   - Schedule second .then() in microtask queue
   - Execute second .then() → Output: **4**
6. Microtask queue empty, get next macrotask:
   - Execute setTimeout callback → Output: **2**

**Q2: What is the difference between microtasks and macrotasks?**

**A:**

**Microtasks:**
- Higher priority, execute before next macrotask
- ALL microtasks execute together
- Sources: Promise callbacks, queueMicrotask, async/await, MutationObserver
- Can starve macrotask queue if endlessly spawned

**Macrotasks:**
- Lower priority, ONE executes per event loop iteration
- Sources: setTimeout, setInterval, I/O, UI events, script loading
- Cannot block microtasks

**Execution order:**
```
Synchronous Code → ALL Microtasks → ONE Macrotask → Repeat
```

**Example:**
```javascript
setTimeout(() => console.log('Macro 1'), 0);
setTimeout(() => console.log('Macro 2'), 0);

Promise.resolve().then(() => console.log('Micro 1'));
Promise.resolve().then(() => console.log('Micro 2'));

// Output: Micro 1, Micro 2, Macro 1, Macro 2
```

**Q3: How does async/await interact with the event loop?**

**A:**

`async/await` is syntactic sugar over promises, so it follows microtask rules:

```javascript
async function example() {
  console.log('1: Before await');
  await Promise.resolve();
  console.log('2: After await');
}

console.log('3: Before example()');
example();
console.log('4: After example() call');

// Output: 3, 1, 4, 2
```

**How it works:**
1. `example()` called, executes synchronously until `await`
2. `await` suspends function, schedules continuation as microtask
3. Returns control to caller
4. After synchronous code completes, microtasks execute
5. Continuation resumes with result of awaited promise

**Equivalent promise code:**
```javascript
function example() {
  console.log('1: Before await');
  return Promise.resolve().then(() => {
    console.log('2: After await');
  });
}
```

**Q4: What is microtask queue starvation? How can it occur?**

**A:**

**Microtask queue starvation** occurs when microtasks continuously spawn more microtasks, preventing macrotasks from ever executing.

**Example:**
```javascript
setTimeout(() => console.log('Never executes'), 0);

function infiniteMicrotasks() {
  Promise.resolve().then(() => {
    console.log('Microtask');
    infiniteMicrotasks(); // Spawns another microtask!
  });
}

infiniteMicrotasks();
// Output: Microtask, Microtask, Microtask... (infinite)
// setTimeout NEVER runs
```

**Why?** The event loop processes ALL microtasks before moving to the next macrotask. If microtasks keep spawning, the macrotask queue never gets serviced.

**Prevention:**
- Limit microtask chains
- Use setTimeout to break into macrotasks
- Monitor microtask queue depth

**Fixed example:**
```javascript
function controlled(depth = 0) {
  if (depth > 100) {
    // Break into macrotask after 100 iterations
    setTimeout(() => controlled(0), 0);
    return;
  }
  
  Promise.resolve().then(() => {
    console.log('Microtask', depth);
    controlled(depth + 1);
  });
}
```

**Q5: How does browser rendering fit into the event loop?**

**A:**

Browser rendering happens AFTER all microtasks, BEFORE the next macrotask:

```
Macrotask → ALL Microtasks → Rendering → Next Macrotask
```

**Example showing blocked render:**
```javascript
button.onclick = () => {
  button.textContent = 'Loading...'; // DOM updated
  
  // 1000 microtasks scheduled
  for (let i = 0; i < 1000; i++) {
    Promise.resolve().then(() => {
      Math.random(); // Some work
    });
  }
  
  Promise.resolve().then(() => {
    button.textContent = 'Done!';
  });
  
  // User NEVER sees "Loading..." because rendering happens
  // AFTER all microtasks (including changing to "Done!")
};
```

**Solution: Use setTimeout to allow render:**
```javascript
button.onclick = () => {
  button.textContent = 'Loading...';
  
  setTimeout(() => {
    // Expensive work
    for (let i = 0; i < 1000000; i++) Math.random();
    button.textContent = 'Done!';
  }, 0);
  
  // Render happens after onclick handler but before setTimeout
  // User sees "Loading..." → "Done!"
};
```

**Q6: What's the difference between setTimeout and setImmediate in Node.js?**

**A:**

**setTimeout:**
- Schedules callback in **timers phase** of event loop
- Minimum delay (1ms+)
- Executes after specified delay

**setImmediate:**
- Schedules callback in **check phase** of event loop
- No delay, executes after poll phase
- More predictable for I/O callbacks

**Example:**
```javascript
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));
```

**Output is non-deterministic** when called in main module because:
- If event loop starts, setTimeout might execute first (depends on system clock)
- If I/O callback context, setImmediate always executes first

**Inside I/O callback (deterministic):**
```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('setTimeout'), 0);
  setImmediate(() => console.log('setImmediate'));
});

// Output: setImmediate, setTimeout (always)
```

**Best practice:** Use setImmediate in Node.js for I/O-related callbacks.

**Q7: How does process.nextTick() differ from microtasks in Node.js?**

**A:**

`process.nextTick()` has HIGHER priority than microtasks:

**Execution order:**
```
Synchronous Code → process.nextTick() → Microtasks → Macrotasks
```

**Example:**
```javascript
console.log('1');

process.nextTick(() => console.log('2: nextTick'));

Promise.resolve().then(() => console.log('3: Promise'));

setTimeout(() => console.log('4: setTimeout'), 0);

console.log('5');

// Output: 1, 5, 2, 3, 4
```

**When to use each:**

**process.nextTick():**
- Execute callback before event loop continues
- Initialize async operations
- Emit events after constructor returns

**Microtasks (Promises):**
- Standard async operations
- Better cross-platform compatibility

**Warning:** process.nextTick() can also starve the event loop:
```javascript
function infiniteNextTick() {
  process.nextTick(infiniteNextTick);
}
infiniteNextTick(); // Blocks event loop forever
```

**Q8: How would you implement a function that executes callback after all pending microtasks?**

**A:**

Use `queueMicrotask()` or `Promise.resolve().then()`:

```javascript
function afterMicrotasks(callback) {
  queueMicrotask(callback);
}

// Or with Promise:
function afterMicrotasks(callback) {
  Promise.resolve().then(callback);
}

// Usage:
console.log('1: Sync');

Promise.resolve().then(() => console.log('2: Microtask'));

afterMicrotasks(() => console.log('3: After microtasks'));

console.log('4: Sync');

// Output: 1, 4, 2, 3
```

**For executing AFTER ALL current microtasks:**
```javascript
function afterAllMicrotasks(callback) {
  Promise.resolve().then(() => {
    Promise.resolve().then(callback);
  });
}

Promise.resolve().then(() => console.log('1'));
afterAllMicrotasks(() => console.log('2'));

// Output: 1, 2 (executes in next microtask "tick")
```

## Key Takeaways

- JavaScript is **single-threaded**; the **event loop** coordinates asynchronous execution
- **Execution order:** Synchronous Code → ALL Microtasks → ONE Macrotask → Repeat
- **Microtasks** (promises, queueMicrotask) have higher priority than **macrotasks** (setTimeout, I/O)
- ALL microtasks execute together before the next macrotask
- **Microtask queue starvation** can occur if microtasks endlessly spawn more microtasks
- Browser **rendering** happens after microtasks, before next macrotask
- **async/await** uses microtasks under the hood
- **setTimeout(fn, 0)** is NOT 0ms; browsers enforce ~4ms minimum
- Node.js has additional APIs: **process.nextTick()** (highest priority), **setImmediate()** (check phase)
- Understanding task queues is crucial for performance and preventing UI blocking

## Resources

- [HTML Spec - Event Loop Processing Model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- [Node.js Documentation - Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Jake Archibald - Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [MDN - Microtasks](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [MDN - Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
- [libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html)
- [Philip Roberts - What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
